# 0006: Deployment & CI/CD (as implemented)

**Status:** Accepted (2026-09-15) — supersedes
[0005](0005-deployment-and-cicd.md)

## Context
ADR 0005 described a target setup (separate secret-free `ci.yml` in both
repos, deploy only on push, Node 24 images, `VITE_API_URL` as a required
client build arg, migrations at container start). What actually got
shipped and is **verified working end-to-end** differs from it in most of
those points. This ADR records the real, working state so agents stop
relying on 0005, and lists the known gaps of that state.

Verified 2026-09-15:
- `chor-app-server`: GitHub Actions run `35014406608` (commit `ad9d163`),
  `build` + `deploy-to-server` green.
- `chor-app-client`: run `35015369719` (commit `5428b43`) green.
- Live: `https://chorapp.wald.pro` serves the new bundle (contains
  `chorappserver.wald.pro`); `https://chorappserver.wald.pro/auth/me`
  answers `401` JSON with
  `access-control-allow-origin: https://chorapp.wald.pro`.

## Decision

### Topology
One VPS runs everything with Docker + the Compose V2 plugin:

| What | Container | Host dir | Host port | Public URL |
|---|---|---|---|---|
| Client (nginx, static Vite bundle) | `chor_app_client` | `/opt/chor_app_client/` | `3012` → 80 | `https://chorapp.wald.pro` |
| Server (NestJS) | `chor_app_serv` | `/opt/chor_app_serv/` | `5050` → 5050 | `https://chorappserver.wald.pro` |
| PostgreSQL | separate container, reached via `host.docker.internal:5432` | — | `5432` | — |

TLS termination and the domain → port mapping happen in a reverse proxy
on the host. **Its config is not in any of the three repos**, and neither
is the Postgres container — both are managed by hand on the host.

### Pipeline (same shape in both repos)
One workflow file per repo, triggered by `push` and `pull_request` to
`main`:

1. **`checks`** job — runs on both PRs and pushes, needs no secrets.
2. **Deploy job(s)** — `needs: checks` and
   `if: github.event_name == 'push'`, so a PR never deploys and a push only
   deploys when the checks are green. Steps: `docker build` on the runner →
   `docker save` to a `.tar` → `scp` the tar + `docker-compose.yml` to the
   host dir → over SSH, a remote script starting with `set -e` (any failing
   command fails the job).

Per repo:
- Client: `.github/workflows/ci.yml`.
  `checks`: `npm ci`, `lint`, `format:check`, `type-check`, `test`,
  `build` (Node 24). Deploy job `build-and-deploy`: `docker load` →
  `docker compose down` → `docker compose up -d` → `docker ps | grep`,
  plus a "Verify deployment" step printing `docker ps` + last 20 log lines.
- Server: `.github/workflows/docker-image.yml`.
  `checks`: `npm ci`, `prisma generate`, `lint:check` (eslint incl.
  prettier, no `--fix`), `test` (unit tests only — the e2e suite boots the
  full `AppModule` and needs a DB), `build` (Node 20, same major as the
  image). Deploy jobs `build` → `deploy-to-server`: `docker load` →
  **`docker compose run --rm -T server npx prisma migrate deploy`** (new
  image, old container still serving) → `docker compose down` →
  `docker compose up -d` → after 15 s, fail with the last 50 log lines
  unless the container is `running` with `RestartCount` 0 → remove tar,
  `docker image prune -f`.
- GitHub secrets, same names in both repos: `SSH_PRIVATE_KEY`,
  `SERVER_USER`, `SERVER_IP`.

Failure behaviour of the server deploy, verified locally with the exact
remote script against a throwaway Postgres (2026-09-15): migration can't
run → job fails, old container keeps running; app crashes on start
(missing `.env` key) → job fails with the app's logs; valid config →
migrations applied, container up, API answers.

### Images
- Client `Dockerfile`: `node:23-alpine3.20` builder, `npm install
  --legacy-peer-deps` (from `package.json` only, no lockfile), `npx vite
  build` → `nginx:alpine` with `nginx.conf` (SPA fallback `try_files $uri
  /index.html`).
- Server `Dockerfile`: single stage `node:20-slim` + `postgresql-client` +
  `openssl`, `npm install`, `npx prisma generate`, `npm run build`
  (`nest build`), `CMD ["node", "dist/main"]`. The image keeps
  devDependencies, so the `prisma` CLI is available for the deploy's
  migration step. The client's `docker-compose.yml` has
  `restart: unless-stopped`, same as the server's.

### Configuration
- **Server:** runtime config comes from `/opt/chor_app_serv/.env` on the
  host (`env_file: .env` in `docker-compose.yml`; compose also forces
  `PORT=5050`, `NODE_ENV=production`). That file is maintained by hand and
  never goes through git or CI; `.dockerignore` keeps any local `.env` out
  of the image. Keys: see `chor-app-server/.env.example`.
- **Client:** there is **no `.env` on the host and no build arg**. The API
  URL is resolved in the browser by `src/utils/apiConfig.ts` from
  `window.location.hostname` (`prodApiUrls`: `chorapp.wald.pro` →
  `https://chorappserver.wald.pro`); only dev/tests read `VITE_API_URL`.
  An unmapped host makes every request fail with an explicit `HttpError`
  rather than hitting nginx's `index.html` fallback. This replaces 0005's
  rejected-alternative reasoning: the failure is loud (a visible error per
  request), not silent, and no per-environment build is needed.
- **CORS:** `chor-app-server/src/main.ts` allows `https://chorapp.wald.pro`
  and any `http://localhost:*`. A new client domain must be added in
  **both** `prodApiUrls` and `allowedOrigins`.

### Server build fix
`tsconfig.json` no longer sets `incremental: true`. With it, `nest build`
(`deleteOutDir: true`) deleted `dist/` but kept `tsconfig.tsbuildinfo`, so
`tsc` considered the output up to date and emitted nothing — the "`dist/
main.js` missing" problem 0005 left open. Verified: two consecutive
`npm run build` runs now both leave `dist/main.js` in place.

## Consequences
- Every green push to `main` in either repo is a production deploy. Lint,
  format, type and unit-test failures block it; anything they don't cover
  (e2e behaviour, the Docker build itself on a PR) doesn't.
- Migrations are applied automatically on every server deploy, **before**
  the new container replaces the old one — so the running (old) code must
  tolerate the new schema for a moment. Write migrations
  backward-compatible (add first, drop in a later release).
- A server deploy whose migration fails leaves production on the previous
  version with the job red; a deploy whose app fails to start leaves
  production **down** with the job red (the old container is already
  removed) — fix forward or redeploy the previous commit.
- Changing server env config means editing `/opt/chor_app_serv/.env` on
  the host and restarting the container; changing the client's API target
  means a code change in `apiConfig.ts` and a redeploy.

## Fixed on 2026-09-15 (were known issues 1–5)
- PRs deployed to production (both workflows triggered deploys on
  `pull_request`) → deploy jobs now `if: github.event_name == 'push'`.
- No quality gate → `checks` job in both repos; server eslint errors in
  `src/main.ts` (untyped CORS callback) fixed, client files formatted.
- Migrations weren't applied on deploy → `prisma migrate deploy` step.
- Client deploy could fail silently (`... && down || true && up` swallowed
  a failed `docker load`) → `set -e` remote scripts in both repos.
- Client container had no restart policy → `restart: unless-stopped`.

## Known issues (ordered by impact)
1. **EOL runtimes:** Node 23 and Alpine 3.20 (client builder), Node 20
   (server; npm already warns `EBADENGINE` for `@prisma/streams-local`,
   which needs Node ≥ 22).
2. **Non-reproducible installs:** client ignores `package-lock.json` and
   uses `--legacy-peer-deps`; server uses `npm install` instead of
   `npm ci`. Production can get different dependency versions than local.
3. **Client Dockerfile obscures build errors:** `vite build || (vite build
   | grep "failed to resolve")` reruns the build and can exit 0, failing
   later at `COPY --from` with a misleading "dist not found".
4. **SSH host key is never verified:** client uses
   `StrictHostKeyChecking=no`, server runs `ssh-keyscan` on every run.
   Store the host's `known_hosts` line in a secret instead.
5. **Stale index.html after deploy:** `nginx.conf` sends no
   `Cache-Control`; browsers may heuristically cache `index.html`, request
   old hashed assets that the new image no longer has, and get
   `index.html` back as JS (SPA fallback) → blank page until reload.
   Fix: `no-cache` for `index.html`, long-lived cache for `/assets/`, and
   a real 404 for missing assets.
6. **Leftovers from other projects:** workflow name "Smart Soft System",
   compose service `chsm_client` + redundant `command:`; server deploy
   creates `data/` and `backups_history/` with `chmod 777` although no
   volume mounts them.
7. **Housekeeping:** client deploy never prunes images or deletes the
   tar (disk fills over time); both deploys do `down` then `up` (short
   downtime); server image is single-stage and ships sources (its
   devDependencies are needed now — the migration step uses the `prisma`
   CLI); GitHub Actions warns that `checkout@v4`, `setup-node@v4`,
   `setup-buildx-action@v3`, `ssh-agent@v0.9.0` run on deprecated Node 20.
8. **To check on the host:** ports are published on `0.0.0.0` (`3012`,
   `5050`). If the firewall doesn't block them, both apps are reachable
   over plain HTTP around the TLS proxy. (Not reachable via the domain
   name from outside on 2026-09-15; the server IP itself was not tested.)
   Binding to `127.0.0.1:PORT:PORT` would close this regardless.

## Open follow-ups
- Optional: GitHub branch protection on `main` requiring the `checks` job,
  so a red PR can't be merged at all (today it only can't deploy).
- Put the reverse-proxy and Postgres container setup under version
  control (or at least document them), so the host can be rebuilt.
- A post-deploy check that the app is *healthy* (HTTP request), not just
  that the container is running — needs a health endpoint in the server.
