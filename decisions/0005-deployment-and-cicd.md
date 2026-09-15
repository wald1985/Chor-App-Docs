# 0005: Deployment & CI/CD

**Status:** Superseded by [0006](0006-deployment-as-implemented.md)
(2026-09-15) — the setup below was never deployed as described; read 0006
for the real, working pipeline.

## Context
Both `chor-app-server` and `chor-app-client` had Docker/CI-CD files sitting
on disk, but an audit found none of it actually worked, and most of it had
never been pushed:

- `chor-app-server`: `Dockerfile` and `docker-compose.yml` existed locally
  but were **not tracked in git at all** — `origin/main` has neither file.
  The committed `.github/workflows/docker-image.yml` builds and deploys a
  Docker image on every push *and every pull request* to `main`, with no
  distinction between the two — a PR alone would trigger a production
  redeploy. The Dockerfile that did exist ran the app via `bun
  src/main.ts` (raw TypeScript, no `nest build`), with no `bun.lock`
  committed anywhere (only `package-lock.json`), so `bun install
  --frozen-lockfile` in CI would fail outright, and this contradicted
  `package.json`'s own `start:prod` script (`node dist/main`) — the only
  production entrypoint actually exercised during development.
- `chor-app-client`: `Dockerfile`/`docker-compose.yml` are tracked but
  **committed empty** (0 bytes) — never actually filled in and pushed. The
  content that did exist locally referenced a missing `nginx.conf` (hard
  build failure), masked real build failures behind `... || (npx vite
  build 2>&1 | grep "failed to resolve")` (grep matching the error text
  makes the step exit 0 even though the build failed), never set
  `VITE_API_URL` (Vite bakes `VITE_*` vars in at build time — omitting it
  silently ships a bundle pointed at `localhost`), and `docker-compose.yml`
  was leftover boilerplate from an unrelated project (`chsm_admin_super`
  naming — matches stray "CHSM" naming found elsewhere, e.g. an earlier
  Swagger title in the server). The committed CI workflow
  (`.github/workflows/webpack.yml`) was GitHub's untouched default
  "Node.js with Webpack" template — this project uses **Vite**, not
  webpack, and `webpack` isn't even a dependency, so this workflow never
  validated a real build.

## Decision

### Server runtime: Node + `nest build` output, not Bun
`chor-app-server`'s Docker image now builds via `npm ci` → `npx prisma
generate` → `npm run build` (two-stage, `node:24-slim`), and runs the
compiled `dist/main.js` with plain Node — matching `start:prod` and
everything actually exercised during development (including native
addons like `bcrypt`, never verified under Bun). Migrations run at
container start (`npx prisma migrate deploy && node dist/main.js`) so a
deploy always leaves the schema in sync before the app accepts traffic.
The final image keeps the full `node_modules` (including `prisma`, a
devDependency) rather than a pruned prod-only install, specifically so
`prisma migrate deploy` has the CLI available at runtime — a deliberate
simplicity-over-image-size tradeoff for a single small service.

### Client: Docker + nginx, `VITE_API_URL` required at build time
`chor-app-client`'s image builds the Vite app and serves the static
output via `nginx:alpine` with an SPA fallback (`nginx.conf`: unmatched
paths → `index.html`, so client-side routing works on a hard refresh).
`VITE_API_URL` is a required Docker build `ARG` — `docker compose build`
fails loudly (`VITE_API_URL must be set`) rather than silently baking in
a wrong or placeholder API URL, since there is no way to change a
Vite-built bundle's API target after the image exists.

### CI (validate) is separate from CD (deploy), and CD never runs on a PR
Each repo now has a `ci.yml` — install, lint, type-check, build, test —
that runs on every push *and* pull request to `main`, needs no secrets,
and deploys nothing. `chor-app-server` additionally has
`docker-image.yml`, which builds the Docker image and deploys it over SSH
to the production host — but its trigger is **`push` to `main` only**;
the pull-request trigger was removed specifically to close the
"opening a PR redeploys production" hole. `chor-app-client` has no
deploy-to-server job yet (see Open follow-ups) — CI there only validates
the build.

### `.dockerignore` in both repos
Excludes `.env` (and `.env.*`, keeping `.env.example`), `node_modules`,
`.git`, `dist`/`coverage`. Without it, `COPY . .` would happily bake a
developer's real `DATABASE_URL`/`JWT_SECRET`/SMTP password (server) into
an image layer if they ever ran `docker build` locally with a `.env`
sitting in the directory.

## Consequences
- Every file this ADR describes must actually be **committed and
  pushed** — none of the working versions existed on `origin/main` before
  this change; a deploy attempted against the previous `main` would have
  failed at the very first Docker-build step (server) or shipped a build
  with no API connectivity (client, once its Dockerfile stopped being
  empty).
- `chor-app-server`'s `npm start` / `npm run start:dev` (bare `nest
  start`, with or without `--watch`) were found to be **unreliable** in
  this environment — the compiler sometimes reports success without
  `dist/main.js` actually existing afterward, crashing the subsequent
  `node dist/main` launch with `MODULE_NOT_FOUND`. Neither `deleteOutDir:
  false` nor forcing `"builder": "tsc"` in `nest-cli.json` (now set,
  since it removes ambiguity either way) made this fully reliable.
  **`npm run build && npm run start:prod` (or the Docker image, which
  uses the same path) is the only combination verified reliable across
  repeated cold-start attempts** — use it whenever a run needs to
  actually work, not just `npm start` for a quick check. Neither Docker
  nor CI use the unreliable scripts, so deploys are unaffected; this is a
  local-dev-convenience issue, tracked as an open follow-up, not blocking.
- CI now actually gates on lint/type-check/test/build for both repos,
  which surfaced (and required fixing, as part of landing this ADR) two
  pre-existing issues that predate it: a `typescript-eslint` deprecation
  in `chor-app-client/eslint.config.js`, and `chor-app-server`'s
  `tsconfig.json` not covering `test/**/*` (its e2e spec was invisible to
  typed linting) combined with a redundant `rootDir` that conflicted with
  including it — both fixed as part of this change, not left for CI to
  discover red.

## Alternatives considered
- **Keep Bun for the server** — rejected: no lockfile ever existed for
  it (only `package-lock.json`), so CI's `bun install --frozen-lockfile`
  was guaranteed to fail on the very next run regardless of anything
  else; native dependencies (`bcrypt`) were never validated under Bun;
  and it contradicted the one production path (`node dist/main`) that
  had actually been exercised throughout development.
- **Default `VITE_API_URL` to a guessed production domain** — rejected:
  a wrong guess baked into a static bundle fails silently (the deployed
  site just can't reach its API, with no error until someone notices);
  failing the build loudly when unset is safer than guessing.
- **Keep `deleteOutDir` disabled to sidestep the `nest start` bug** —
  tried, didn't reliably fix it (see Consequences); reverted to `true`
  (the more conventional setting) since it wasn't the actual fix.

## Open follow-ups
- `chor-app-client` has no deploy-to-server CI job — needs a target host
  decision (same VPS as the server? a separate one? a static host
  instead of Docker+nginx entirely?) before one can be wired up the way
  `chor-app-server`'s `docker-image.yml` already is.
- The `nest start`/`nest start --watch` reliability issue itself is
  unresolved (workaround documented above, root cause not fully
  isolated) — worth a focused investigation if it becomes disruptive
  enough to matter, not blocking anything today.
- `prisma migrate deploy` currently runs as part of the container's own
  startup command. Fine for a single-instance deploy; would need to move
  to a distinct release step if this ever becomes a multi-instance
  deployment (to avoid concurrent migration attempts).
