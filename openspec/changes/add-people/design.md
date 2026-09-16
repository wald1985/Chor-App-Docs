## Context

Legacy (`chor-app_v3.html`, tab *Verwaltung*): two independent localStorage
string arrays `pianisten` / `dirigenten`, case-insensitive duplicate check
per list, removal splices the name while past log entries keep the text.
`domain-model.md` already decided one `Person` entity shared by both roles,
separate from `User`, scoped per Community. Community targeting and the
`PEOPLE_MANAGE` permission come from `add-community-access-control`
(ADR 0007). Product decisions (2026-09-16): roles combinable, hardcoded for
MVP; removal = archive; Person↔User link deferred.

## Goals / Non-Goals

**Goals:**
- Stable `personId` other contexts reference (Rehearsal Log, Performance
  Log, Absences, Reporting) without depending on People internals.
- Second concrete instance of the ADR 0002 module layout, first one using
  the Community guard.

**Non-Goals:** Person↔User link, absences, legacy import, statistics,
contact details (phone/email) — not in the legacy app, add when needed.

## Decisions

### Roles: hardcoded enum set on the aggregate
```prisma
enum PersonRole { PIANIST CONDUCTOR }
model Person {
  id          String       @id @default(uuid())
  communityId String
  name        String
  nameKey     String       // lower(trim(name)), for uniqueness
  roles       PersonRole[]
  archivedAt  DateTime?
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
  community   Community    @relation(fields: [communityId], references: [id], onDelete: Cascade)
  @@unique([communityId, nameKey])
  @@index([communityId])
  @@map("people")
}
```
- Enum array instead of a `PersonRoleAssignment` join table: roles are a
  fixed tiny set with no attributes of their own, so a join table adds
  nothing in the MVP. Filtering uses `roles: { has: 'PIANIST' }`. If roles
  ever become configurable or get attributes (e.g. "since"), migrate to a
  table.
- Rejected: two booleans `isPianist`/`isConductor` — every new role would be
  a schema + API change; the enum set keeps the API shape (`roles: [...]`)
  stable.
- Rejected: separate `Pianist`/`Conductor` entities — exactly the legacy
  duplication problem.

Domain: `Person` aggregate with `PersonName` value object (trim, 1–100,
`key` = lowercase) and `PersonRoles` value object (non-empty, deduplicated
set). Methods: `rename`, `changeRoles`, `archive`, `restore`, `isArchived`,
`hasRole`. Invariant "at least one role" and "archived → not editable" live
in the aggregate.

### Uniqueness: DB constraint + domain pre-check
`@@unique([communityId, nameKey])` is the real guarantee (races); the use
case pre-checks to return a friendly conflict including the existing
Person's id and archived state. Unique constraint violation (P2002) is
mapped to the same domain error. `nameKey` is a stored column rather than a
functional index so Prisma can express the constraint. This resolves the
general "domain vs DB uniqueness" open point for People: **both**.

### Archive instead of delete
`archivedAt` timestamp (null = active). No DELETE endpoint. Archived People
keep uniqueness of their name, so returning members are restored, not
duplicated. Later capabilities that reference a Person must accept archived
ones for existing history but only offer active ones for new entries.
Archive/restore are idempotent (200 with current state).

### API (`src/people/interface/people.controller.ts`)
All under `/communities/:communityId/people`, guarded by `JwtAuthGuard` +
`CommunityMemberGuard`:

| Method & path | Permission | Notes |
|---|---|---|
| `GET /` | member | `?role=PIANIST\|CONDUCTOR`, `?includeArchived=true`; sorted by `nameKey` |
| `GET /:personId` | member | 404 if not in this Community |
| `POST /` | `PEOPLE_MANAGE` | `{ name, roles }` → 201 |
| `PATCH /:personId` | `PEOPLE_MANAGE` | `{ name?, roles? }`; 409 if archived |
| `POST /:personId/archive` | `PEOPLE_MANAGE` | idempotent |
| `POST /:personId/restore` | `PEOPLE_MANAGE` | idempotent |

Response: `{ id, name, roles, archived, archivedAt, createdAt, updatedAt }`.
Errors: validation 400, forbidden 403, not found 404, name conflict 409
(`{ existingPersonId, existingArchived }`), archived edit 409.

No pagination: a choir has tens of People.

### Module boundaries
`PeopleModule` imports `IdentityModule` only for the guard/decorators. It
exports nothing yet; future contexts that must validate a `personId` will
get a small read port (`PersonLookup`) exported by People rather than
querying the `people` table directly.

## Risks / Trade-offs

- **Name-only identity for humans with the same name** → uniqueness forces
  distinct labels ("Anna K.", "Anna M."), same as legacy. Acceptable.
- **Archived names block reuse** → intentional; conflict response points to
  restore.
- **Enum array migration later** → small, mechanical if ever needed.

## Migration Plan

Additive migration (new enum + table). Legacy names are imported later by
Legacy Import; merging a name present in both legacy lists into one Person
with both roles is noted for that capability.

## Open Questions

- Person ↔ User link (deferred).
- Whether the German UI label is "Personen" or "Mitwirkende" — decide when
  building the client screen (UI-only).
