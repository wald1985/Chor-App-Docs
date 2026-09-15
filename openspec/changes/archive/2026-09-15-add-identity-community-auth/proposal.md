## Why

`chor-app-server` had no way for a user to create an account or authenticate,
and no database connection at all. Every other bounded context needs a
`communityId` to scope its data to (ADR 0003), and nothing can be built
behind an authenticated endpoint until registration, login, and a way to
identify the calling user exist. This change implements that foundation —
Identity & Community persistence via Prisma/PostgreSQL, plus the concrete
auth mechanism that ADR 0003 and `domain-model.md` had explicitly left open.

## What Changes

- Decide and implement the concrete auth mechanism left open by ADR 0003:
  **email + password**, with a **JWT bearer token** (`Authorization: Bearer
  <token>`) returned on login/register and required on protected endpoints.
  Recorded as `decisions/0004-auth-mechanism.md`.
- Connect `chor-app-server` to PostgreSQL via Prisma (ADR 0002), with the
  `Community` / `User` / `CommunityMembership` models as the first slice of
  the shared `prisma/schema.prisma`.
- Registration: a single request creates a `Community`, a `User`, and a
  `CommunityMembership` with role `ADMINISTRATOR`, in one transaction (per
  ADR 0003 — "registering creates a Community and an Administrator User in
  one step").
- Login: verifies email + password, returns a JWT plus the user's
  `CommunityMembership` list (id, community name, role) so the client can
  drive a Community switcher later.
- A protected `GET /me`-style endpoint that resolves the caller from the JWT
  and returns their profile plus memberships.
- Passwords are hashed (bcrypt), never stored or returned in plain text.

**Not in scope for this change** (left open, tracked in the new ADR and in
`domain-model.md`'s "still open" list):
- Inviting further Users by email (SMTP-based invite flow) — the transport
  (ADR 0002) and the "who can invite" rule (ADR 0003) are decided, but
  token/expiry/resend mechanics are not.
- An "active Community" selector/guard enforced on every request — nothing
  is Community-scoped yet besides Identity itself, so there is no request to
  enforce it on. Revisit when the first Community-scoped capability
  (Repertoire, Rehearsal) is built.
- Role granularity beyond `ADMINISTRATOR`/`MEMBER`.

## Capabilities

### New Capabilities
- `identity/registration-and-login`: account registration (which also
  creates the owning Community), email/password login issuing a JWT, and
  resolving the authenticated user from that token.

### Modified Capabilities
(none — this is the first capability spec for this bounded context)

## Impact

- **chor-app-server**: new `src/identity/` module
  (domain/application/infrastructure/interface, per ADR 0002), new
  `src/shared/prisma/` module, `prisma/schema.prisma` +
  first migration, new dependencies (`@prisma/client`, `prisma`,
  `@prisma/adapter-pg`, `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt`,
  `bcrypt`, `@nestjs/config`, `class-validator`, `class-transformer`).
- **chor-app-client**: no code yet, but future auth screens/HTTP calls must
  send the JWT as `Authorization: Bearer <token>` (not a cookie) per
  `decisions/0004-auth-mechanism.md`.
- New env vars: `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`.
