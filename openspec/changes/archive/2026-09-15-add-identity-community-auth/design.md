## Context

`chor-app-server` was an empty NestJS scaffold with no persistence and no
auth. ADR 0002 already fixed the stack (NestJS + Prisma + PostgreSQL, DDD
per bounded context) and ADR 0003 already fixed the tenancy model
(Community as tenant root, multi-Community membership, registration creates
a Community + Administrator in one step). What ADR 0003 explicitly left
open was the auth mechanism itself and how a session is carried — both had
to be decided as part of implementing this capability. See proposal.md -
Why.

## Goals / Non-Goals

**Goals:**
- Stand up the Identity & Community bounded context as the first concrete
  instance of the ADR 0002 module layout (domain / application /
  infrastructure / interface), so later bounded contexts have a real
  precedent to follow.
- Decide and record the auth mechanism (this design doc's output feeds
  `decisions/0004-auth-mechanism.md`).
- Get `chor-app-server` connected to the real PostgreSQL instance.

**Non-Goals:**
- Community invites by email — transport is decided (ADR 0002/nodemailer),
  but token/expiry/resend mechanics are not; deferred to its own capability.
- Enforcing an "active Community" on every request — there is nothing
  Community-scoped yet besides Identity itself.
- Role granularity beyond `ADMINISTRATOR`/`MEMBER`.

## Decisions

### Auth mechanism: email + password, JWT bearer token
Chosen over magic-link and OAuth. Rationale: the product already has a
decided, required email-based invite flow (ADR 0003), so email deliverability
isn't a blocker either way, but password login doesn't require SMTP to be
configured for every single login (only for invites/resets) — magic-link
would make SMTP a hard dependency of every login. OAuth was rejected because
it complicates "Administrator creates a Community and invites by email" with
provider-specific account linking, for no requirement that asked for it.

Session is carried as a JWT in `Authorization: Bearer <token>`, not an
httpOnly cookie, even though the scaffold already had `cookie-parser` and
CORS `credentials: true` wired in (unused leftovers from initial setup, not
a decision). Bearer was chosen so the client's fetch-based HTTP utility
(ADR 0001) owns the token explicitly rather than relying on browser cookie
jars, and to avoid CSRF-protection design work that cookie sessions would
require. Recorded as `decisions/0004-auth-mechanism.md`, since this is a
cross-cutting choice future capabilities' protected endpoints all rely on,
not just this one.

### DDD layering inside `src/identity/`
Following ADR 0002 literally:
- `domain/`: `User`, `Community`, `CommunityMembership` entities, a
  `CommunityRole` enum, and **ports** (interfaces) for `UserRepository`,
  `MembershipRepository`, a composite `RegistrationRepository` (create
  Community + User + Membership atomically), `PasswordHasher`, and
  `TokenIssuer`. No Prisma or Nest imports here.
- `application/`: `RegisterUseCase`, `LoginUseCase`, `GetCurrentUserUseCase`
  — orchestrate domain ports only.
- `infrastructure/`: Prisma-backed implementations of each port (the
  registration repository wraps `Community`+`User`+`CommunityMembership`
  creation in one `$transaction`), plus `BcryptPasswordHasher` and a
  `JwtTokenIssuer`/`JwtStrategy` pair.
- `interface/`: `AuthController` (`POST /auth/register`, `POST /auth/login`,
  `GET /auth/me`), DTOs with `class-validator`, a `JwtAuthGuard`, and a
  `@CurrentUser()` param decorator.

A composite `RegistrationRepository` (rather than composing three separate
repository calls from the use case) keeps the `$transaction` boundary inside
`infrastructure/`, so `application/` never has to know about Prisma
transactions.

### Database: Prisma 7 with the `@prisma/adapter-pg` driver adapter
Prisma 7 removed the `url` field from the `datasource` block in
`schema.prisma`; the connection string now lives in `prisma.config.ts` (used
by the CLI for `generate`/`migrate`) and is passed explicitly to
`PrismaClient` via a driver adapter (`@prisma/adapter-pg`) at runtime. Note
for future contexts extending `prisma/schema.prisma`: don't add
`url = env(...)` back into the datasource block, it's rejected by the CLI.

`npm install prisma` currently resolves to a `8.0.0-rc.15` prerelease under
the npm `latest` tag (a preview "agentic" CLI with a different command set).
Pinned to the last stable `7.10.0` explicitly in `package.json` — worth
re-checking when Prisma 8 actually goes GA.

### Login response includes all CommunityMemberships, not just one
Per ADR 0003 a User can belong to multiple Communities with no single
implicit one. Login returns the full membership list (Community id, name,
role) rather than picking one, so the client can build the "active
Community" switcher later. No Community-scoping guard consumes this yet
(see Non-Goals).

## Risks / Trade-offs

- **[Risk]** JWT is a long-lived bearer credential the client must store
  itself (no server-side session invalidation). → **Mitigation**: kept the
  default expiry short-ish (`JWT_EXPIRES_IN=1d`, configurable); revisit
  refresh-token/revocation if a real incident or product requirement calls
  for it — not built speculatively here.
- **[Risk]** `identity/` is the only precedent for the ADR 0002 module
  layout; if it's awkward in practice, every later bounded context copies
  the awkwardness. → **Mitigation**: none needed yet — flagged here so the
  next bounded context's author reviews this one first and raises a change
  if the pattern doesn't hold up.
- **[Trade-off]** The composite `RegistrationRepository` port is
  Identity-specific rather than a generic "unit of work" abstraction. Chosen
  deliberately over building a cross-context transaction abstraction with
  only one caller so far (YAGNI); revisit only if a second bounded context
  needs the same atomic-multi-aggregate-create shape.

## Migration Plan

Additive only — first migration on a previously-empty schema
(`communities`, `users`, `community_memberships` tables). No existing data,
no rollback concerns beyond the standard `prisma migrate` down path.
