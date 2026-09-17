# 0011: Superadmin identity and session — separate entity, `aud`-typed JWT, no permission model

**Status:** Accepted (2026-09-17)
**Scope:** chor-app-server (`SuperadminModule`, `IdentityModule` — one check in `JwtStrategy`,
`LibraryModule`/`LibraryAdminModule` — guard swap), `capability-breakdown.md`
**Related:** ADR 0004 (amended — a JWT can now name a `User` or a `Superadmin`, `aud` says which),
ADR 0008, ADR 0010 (superseded: `SuperAdminGuard` placeholder is gone)

## TL;DR for agents
- **`Superadmin` is a separate entity and table**, not a `User` role or Community membership. Own login
  (`POST /admin/auth/login`), own profile (`/admin/me`), own token. A person can have both a `User` account
  and a `Superadmin` account with the same or different email — they're unrelated logins.
- **All superadmins are equal.** No permission/role model between them. Any superadmin can manage any other
  superadmin (create, rename, change email, set password, delete), except: can't delete themselves, and the
  last remaining superadmin can't be deleted (checked under an advisory lock to survive concurrent deletes).
- **Token type is the JWT `aud` claim**, not a separate secret: user tokens carry no `aud`; superadmin tokens
  carry `aud: "chor-app-superadmin"`. Both are signed with the same `JWT_SECRET` — no new env vars.
  `JwtStrategy` (user) rejects a token whose `aud` matches the superadmin value; `SuperadminJwtStrategy`
  requires it. A user token and a superadmin token are never interchangeable in either direction.
- **No forced password reset**, ever (not after seed, not after another superadmin sets your password).
- **No self-service "forgot password"** for superadmins — recovery is only another superadmin (`PUT
  /admin/superadmins/:id/password`) or the seed script's `--reset-password` on the host.
- **First superadmin is created by a script**, not registration: `node dist/superadmin/interface/cli/seed-superadmin.js
  --email <e> --name <n>` (or `--reset-password` for an existing one). Prints a generated password once to
  stdout; never accepted as a CLI argument.
- **`/admin/library/...` (ADR 0010) now requires a superadmin token**; the placeholder `SuperAdminGuard` that
  used to accept any authenticated user is gone. `GET /library/...` accepts a user *or* a superadmin token
  (`UserOrSuperadminAuthGuard`).
- Client (admin panel) is a separate future iteration; this ADR covers the server only.

## Context
Catalog editing (ADR 0010 §6) shipped behind a placeholder guard that accepted any authenticated `User`,
an accepted risk until a real platform-admin identity existed. This ADR introduces that identity: a
superadmin who can manage other superadmins and — by having a valid superadmin token — pass the catalog's
admin guard. Broader superadmin powers (user support, Community management, bans) are explicitly future
work; this iteration only covers superadmin self-management and closing ADR 0010's guard gap.

## Decision

### 1. Entity and table
Own `Superadmin` model (`id`, `email` unique, `name`, `passwordHash`, `tokenVersion`, timestamps) in its own
`superadmins` table — not `users`, not a `CommunityMembership` role. No relations to any other table;
deletion is a plain `DELETE`, nothing cascades (superadmins aren't referenced anywhere).

### 2. Token design
| | User (existing, ADR 0004) | Superadmin |
|---|---|---|
| Sign | `JwtService.sign({ sub, email, tokenVersion })` | same, `+ { audience: 'chor-app-superadmin' }` |
| Secret / expiry | `JWT_SECRET`, `JWT_EXPIRES_IN` (unchanged, no new env vars) | same |
| Verify strategy | Passport `'jwt'`: **now also rejects** a payload whose `aud` is (or contains) the
  superadmin audience | Passport `'superadmin-jwt'`: passport-jwt's own `audience` option requires it |
| `request.user` | `AuthenticatedUser { id, email, name }` | `SuperadminPrincipal { kind: 'superadmin', id, email, name }` |
| Guard | `JwtAuthGuard` | `SuperadminAuthGuard`; `UserOrSuperadminAuthGuard` accepts either |

Both strategies read the fresh `tokenVersion` from the database on every request and reject a mismatch, so a
password change or deletion invalidates outstanding tokens immediately (same pattern as `User`, ADR 0004).

**Implementation pitfall worth flagging for future agents:** `UserOrSuperadminAuthGuard` is
`AuthGuard(['jwt', 'superadmin-jwt'])`. Passport tries strategies in order and only advances to the next one
on a strategy **failure** (`fail()`); if a strategy's `validate()` *throws*, `@nestjs/passport` turns that
into a Passport **error** (`error()`), which aborts the whole chain instead of trying the next strategy. Both
`JwtStrategy.validate` and `SuperadminJwtStrategy.validate` therefore return `null` (never throw) on any
rejection — the guard's default `handleRequest` turns a falsy result into 401 on its own.

### 3. Authorization model
No roles, no permissions table for superadmins. `SuperadminAuthGuard` alone is the only access decision
for `/admin/me`, `/admin/superadmins/...` and (via the catalog's own guard swap) `/admin/library/...`.
Self-protection is at the domain level, not a permission check:
- `Superadmin.assertDeletableBy(actorId)` → `CannotDeleteSelfError` if `actorId === id`.
- `Superadmin.assertPasswordSettableBy(actorId)` → `UseChangePasswordError` if `actorId === id` (use
  `POST /admin/me/change-password` instead, which requires the current password).
- Deleting the last remaining superadmin → `LastSuperadminError`, enforced inside a transaction holding a
  Postgres advisory lock (`pg_advisory_xact_lock`) around a `findUnique` + `count()` + `delete`, so two
  superadmins deleting each other at the same instant can't both succeed and leave zero.

### 4. Password handling
- Bcrypt, same cost factor as `User` (10 rounds). Own `PasswordHasher` port/adapter in `SuperadminModule`
  (not imported from `IdentityModule` — ADR 0008 boundaries; Identity has no barrel to import it through, and
  the adapter is ~15 lines).
- Login compares against a precomputed dummy bcrypt hash when the email is unknown, so response time doesn't
  reveal whether an email is registered (same technique available to, though not required of, `User` login).
- No forced rotation. No self-service reset by email — recovery is always another human action (another
  superadmin, or the operator running the seed script on the host).

### 5. Seed script
`src/superadmin/interface/cli/seed-superadmin.ts`, compiled by the normal `nest build` into
`dist/superadmin/interface/cli/seed-superadmin.js` (not a `ts-node` dev script — runs in the production image
without dev dependencies). `NestFactory.createApplicationContext`, no HTTP server. Generates a 24-character
`base64url` password via `crypto.randomBytes`, prints it to stdout exactly once, never accepts a password as
an argument (keeps it out of shell history and `ps`). `--reset-password` bumps `tokenVersion`, invalidating
any existing session for that superadmin.

### 6. Catalog guard swap (ADR 0010)
`src/library-admin/interface/guards/super-admin.guard.ts` (the placeholder) is deleted.
`AdminSeriesController`/`AdminBooksController`/`AdminSongsController`/`AdminThemesController`/
`AdminImportsController` now carry `@UseGuards(SuperadminAuthGuard)` instead of
`@UseGuards(JwtAuthGuard, SuperAdminGuard)`. `LibraryController` (`GET /library/...`) carries
`@UseGuards(UserOrSuperadminAuthGuard)` instead of `@UseGuards(JwtAuthGuard)`. `LibraryModule` and
`LibraryAdminModule` import `SuperadminModule` (for the guards) instead of importing `IdentityModule`
directly for that purpose. No cycle: `SuperadminModule` only imports `IdentityModule` and knows nothing
about the catalog.

## Consequences
- Capability **#11 Superadmin**: server implemented; client (admin panel) is separate future work.
- ADR 0010's accepted risk ("anyone with a JWT can write the catalog") is closed: `/admin/library` now
  requires a real superadmin token. A regular `User` token gets 401 there where it previously got 2xx.
- ADR 0004 is amended: a bearer token names either a `User` or a `Superadmin`; `aud` says which. Passport
  strategy authors elsewhere in this codebase should remember the fail-vs-error distinction above before
  adding a third strategy to any multi-strategy guard.
- Until an operator runs the seed script on a given deployment, no superadmin exists there and `/admin/me`,
  `/admin/superadmins` and `/admin/library` all answer 401 for everyone.

## Open questions
None blocking — OQ-2 (the `aud` value, `"chor-app-superadmin"`) is closed by this ADR.

## Alternatives considered
- **Role or flag on `User`** — mixes a platform-level identity into Community-tenant data; rejected per the
  owner's explicit instruction that Superadmin is a separate account.
- **Separate `SUPERADMIN_JWT_SECRET`** — would need a new env var on every deployment (`docker-compose.yml`,
  host `.env`) for no real isolation benefit over a distinguishing `aud` claim that passport-jwt already
  knows how to verify.
- **Permission levels between superadmins** — deferred; at the expected scale (≤ 20 trusted operators) a
  binary "is a superadmin" is enough, and levels can be added later as a new migration without breaking
  this design.
- **Self-service password reset by email for superadmins** — would need a token table and email flow
  mirroring `User`'s; the owner decided recovery-by-another-superadmin-or-script is sufficient at this scale.
