## Context

Identity & Community is implemented (`identity/registration-and-login`,
`identity/password-management`): JWT bearer auth (ADR 0004), `User`,
`Community`, `CommunityMembership` with role `ADMINISTRATOR | MEMBER`
(ADR 0003). Nothing Community-scoped exists yet, so there is no mechanism to
target a Community or authorize an action inside one. People is the first
consumer; Repertoire, Rehearsal Log, Performance Log and Absences will need
the same mechanism. Product decision (2026-09-16): members get individual
permissions set by the Administrator; keep everything hardcoded for the MVP.

## Goals / Non-Goals

**Goals:**
- One reusable way for any bounded context to say "this endpoint belongs to
  Community X and requires permission Y".
- Administrator can grant/revoke permissions per member.
- Keep other bounded contexts decoupled from Identity internals: they use a
  guard + decorators, never Identity repositories.

**Non-Goals:**
- Invites, member removal, role changes (ADMINISTRATOR ↔ MEMBER).
- Custom roles, permission groups, runtime-defined permissions.
- Read permissions: every member can read all data of their Community.
  Permissions only gate *changes*.

## Decisions

### Target Community in the URL path
`/communities/:communityId/<resource>`. Chosen over an `X-Community-Id`
header (hidden state, easy to forget, awkward in logs/caches) and over
embedding the active Community in the JWT (switching would require a new
token; contradicts multi-community membership). The client's "active
Community" is purely client state that builds the URL. → ADR 0007.

### Non-member → 403, not 404
Community ids are UUIDs, so existence leakage is negligible; a uniform 403
is simpler to reason about. Recorded in ADR 0007.

### Permissions: code enum + array column on the membership
```prisma
enum CommunityPermission { PEOPLE_MANAGE }
model CommunityMembership { … permissions CommunityPermission[] @default([]) }
```
A PostgreSQL enum array keeps the MVP to one column instead of a join
table. Adding a permission = add an enum value + migration in the change
that needs it. If permissions ever become runtime-configurable, migrate to a
table then. Names follow `<CONTEXT>_<ACTION>`.

### Effective permissions computed in the domain
`CommunityMembership.effectivePermissions()` returns `ALL_PERMISSIONS` for
`ADMINISTRATOR`, stored set otherwise; `hasPermission(p)` uses it. Stored
permissions on an Administrator are ignored (and cannot be set via API).

### Replace, not add/remove
`PUT …/members/:membershipId/permissions` with `{ permissions: [...] }`
replaces the full set: idempotent, matches a checkbox UI, no partial-update
edge cases.

### Guard design (shared, reusable)
- `CommunityMemberGuard` (runs after `JwtAuthGuard`): reads
  `:communityId`, loads the membership via an Identity application service
  (`ResolveMembershipUseCase`), 403 if none, attaches
  `{ membershipId, communityId, role, permissions }` to the request.
- `@RequirePermission(CommunityPermission.X)` metadata, checked by the same
  guard after resolution.
- `@CurrentMembership()` param decorator for controllers.
- Exported by `IdentityModule`; other modules import `IdentityModule` and use
  these only. Membership is loaded per request (one indexed query on
  `(userId, communityId)`), so permission changes take effect immediately —
  no need to put permissions into the JWT or bump `tokenVersion`.

### Endpoints (in `src/identity/interface/controllers/members.controller.ts`)
- `GET /communities/:communityId/members` — Administrator only.
- `PUT /communities/:communityId/members/:membershipId/permissions` —
  Administrator only; 404 if the membership isn't in that Community; 409 if
  the target is an `ADMINISTRATOR`.
"Administrator only" is a role check in the guard (`@RequireRole(ADMINISTRATOR)`),
not a permission — managing permissions is not itself delegable in the MVP.

## Risks / Trade-offs

- **No members exist until invites ship** → member paths are verified by
  e2e tests that insert a `MEMBER` membership directly.
- **Enum array is harder to query "who has X"** → acceptable, no such
  query is needed yet.
- **Per-request membership lookup** → one indexed query; fine at choir scale.

## Migration Plan

Additive migration: new enum + `permissions` column with default `{}`.
Existing memberships (all Administrators today) need no data change.

## Open Questions

- Whether an Administrator should also be able to delegate "manage
  permissions" — deferred until invites exist.
