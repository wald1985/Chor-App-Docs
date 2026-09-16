## Why

People (`add-people`) is the first capability whose data lives *inside* a
Community rather than being the Community/User itself. Two things are still
missing for any Community-scoped capability:

1. **Which Community a request targets.** `identity/registration-and-login`
   explicitly deferred "an active-Community selector/guard enforced on every
   request … until the first Community-scoped capability is built". That
   moment is now.
2. **Who in a Community may change what.** Only `ADMINISTRATOR`/`MEMBER`
   roles exist. The product owner decided (2026-09-16) that members need
   **individual permissions, granted by the Administrator** — e.g. a member
   may be allowed to manage People without becoming an Administrator.

Both are cross-cutting (every later capability — Repertoire, Rehearsal Log,
Absences — needs the same two checks), so they are delivered as their own
Identity change that `add-people` builds on, instead of being buried inside
People.

## What Changes

- **Community-scoped routes:** every Community-scoped endpoint lives under
  `/communities/:communityId/...`. A reusable guard verifies that the
  authenticated User has a `CommunityMembership` in that Community and
  rejects the request otherwise. Recorded as
  `decisions/0007-community-scoped-requests-and-permissions.md`.
- **Hardcoded permission set:** a `CommunityPermission` enum in code (MVP:
  no user-defined permissions or custom roles). This change introduces the
  mechanism plus the first value, `PEOPLE_MANAGE`, since People is the first
  consumer. Each later capability adds its own value(s) in its own change.
- **Per-membership permissions:** `CommunityMembership` gets a set of granted
  permissions (default: empty). `ADMINISTRATOR` implicitly holds every
  permission; nothing needs to be granted to it.
- **Administrator manages permissions:** list a Community's members with
  their role and permissions; replace a `MEMBER`'s permission set.
- **Permission guard:** a reusable `@RequirePermission(...)` check usable by
  any bounded context's controller.
- `POST /auth/login` and `GET /auth/me` return each membership's
  **effective** permissions so the client can show/hide actions.

**Not in scope:**
- Community invites — until they exist, a Community in practice has only
  its Administrator, so the member-permission endpoints are mainly
  exercised by tests. The model is ready for invited members.
- Promoting/demoting between `ADMINISTRATOR` and `MEMBER`, removing members.
- Custom roles, permission groups, or permissions editable at runtime.

## Capabilities

### New Capabilities
- `identity/community-access`: resolving and authorizing the target
  Community of a request, the hardcoded permission set, and Administrator
  management of member permissions.

### Modified Capabilities
(none — the additive "login and current-user responses include effective
permissions" requirement is specified in `identity/community-access`, so
`identity/registration-and-login` stays unchanged)

## Impact

- **chor-app-server:** `prisma/schema.prisma` (`CommunityPermission` enum,
  `permissions` column on `CommunityMembership`, migration defaulting to
  empty); `src/identity/` (domain, use cases, `MembersController`); new
  shared `CommunityMemberGuard` + `@RequirePermission()` + `@CurrentMembership()`
  exported for other modules.
- **chor-app-client:** later — a members/permissions screen for
  Administrators; hide actions based on returned permissions.
- **chor-app-docs:** ADR 0007, glossary (*Berechtigung* → Permission).
