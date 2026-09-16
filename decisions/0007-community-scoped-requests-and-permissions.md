# 0007: Community-scoped requests and member permissions

**Status:** Accepted (2026-09-16)

## Context
ADR 0003 makes `Community` the tenant root with multi-community membership
and roles `ADMINISTRATOR`/`MEMBER`; ADR 0004 authenticates requests with a
JWT that identifies only the `User`. The first Community-scoped capability
(People, `openspec/changes/add-people`) needs to know which Community a
request targets and whether the caller may change data there. Product
decision: members get individual permissions, set by the Administrator;
everything hardcoded for the MVP.

## Decision

### Target Community is part of the URL path
Every Community-scoped endpoint is routed as
`/communities/:communityId/<resource>...`. A shared `CommunityMemberGuard`
(after `JwtAuthGuard`) loads the caller's `CommunityMembership` for that id
on every request and rejects with **403** if there is none (also when the
Community doesn't exist). The "active Community" is client-side state only.

### Hardcoded permission set on the membership
- `CommunityPermission` is a code-defined enum (Prisma enum), names
  `<CONTEXT>_<ACTION>`; first value `PEOPLE_MANAGE`. Each capability adds its
  own values in its own change.
- `CommunityMembership.permissions` stores the granted set (default empty).
- `ADMINISTRATOR` implicitly holds all permissions; only `MEMBER`
  permissions are stored/settable.
- Permissions gate **changes** only; every member may read all data of the
  Community.
- Only an Administrator can view members and set permissions (role check,
  not delegable in the MVP).
- Controllers declare requirements with `@RequirePermission(...)` /
  `@RequireRole(...)`; other bounded contexts use these exported guards and
  decorators, never Identity repositories.

## Consequences
- Permission changes apply immediately (membership is loaded per request);
  permissions are not embedded in the JWT and don't touch `tokenVersion`.
- One indexed lookup per Community-scoped request.
- Client builds URLs from the active Community and uses the effective
  permissions returned by login / `GET /auth/me` to show or hide actions.
- Adding a permission requires a migration (enum value) — acceptable for a
  hardcoded MVP set.

## Alternatives considered
- **`X-Community-Id` header** — hidden per-request state, easy to omit,
  invisible in URLs/logs/caches.
- **Active Community inside the JWT** — switching Community would need a new
  token; awkward with multi-community membership.
- **More fixed roles instead of permissions** (e.g. `PEOPLE_MANAGER`) —
  combinations explode as capabilities grow; the product owner asked for
  per-member permissions set by the Administrator.
- **Permission table / runtime-defined permissions** — unnecessary for a
  fixed MVP set; revisit if permissions must become configurable.
- **404 for non-members** — ids are UUIDs, existence leakage negligible;
  uniform 403 is simpler.
