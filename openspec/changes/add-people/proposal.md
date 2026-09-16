## Why

Rehearsal Log, Performance Log, Reporting and Absences all refer to the
pianists and conductors of a Community. The legacy app keeps them as two
separate name lists ("Klavierspieler verwalten" / "Dirigenten verwalten",
tab *Verwaltung*), so one human who both plays and conducts appears twice,
and history stores plain text names. The new app needs one `Person` record
per human, with stable identity other capabilities can reference by id.

## What Changes

- New bounded context **People** (`src/people/` in chor-app-server).
- `Person` entity scoped to one Community: name + a set of roles.
- **Roles are combinable:** a Person can be `PIANIST`, `CONDUCTOR`, or both
  (at least one). Roles are a **hardcoded enum** for the MVP — no custom
  roles.
- Any member of the Community can list and view People (optionally filtered
  by role, e.g. for the pianist dropdown).
- Creating, editing (name, roles), archiving and restoring a Person requires
  the `PEOPLE_MANAGE` permission (Administrator implicitly; members if
  granted — see `add-community-access-control`).
- **No hard delete:** removing a Person archives it. Archived People are
  hidden from default lists (and so from future selection dropdowns) but
  remain resolvable by id, so later history/reporting keeps its references.
  Archived People can be restored.
- Name is unique per Community, case-insensitive, including archived People
  (re-adding someone who left means restoring them, not creating a
  duplicate).

**Not in scope:**
- Link Person ↔ User (deferred until invites exist).
- Absences (*Abwesenheiten*) — own capability, references `personId`.
- Importing the legacy name lists — part of Legacy Import.
- Any statistics per pianist/conductor — Reporting.

## Capabilities

### New Capabilities
- `people/person-management`: create, list, view, edit, archive and restore
  People with combinable, hardcoded roles within a Community.

### Modified Capabilities
(none)

## Impact

- **Depends on:** `add-community-access-control` (Community-scoped routes,
  `PEOPLE_MANAGE` permission guard) — must be implemented first.
- **chor-app-server:** `prisma/schema.prisma` (`Person`, `PersonRole` enum),
  migration, new `PeopleModule` (domain/application/infrastructure/interface).
- **chor-app-client:** later — German UI "Personen verwalten" replacing the
  two legacy lists; role checkboxes *Klavierspieler* / *Dirigent*.
- **chor-app-docs:** `domain-model.md` (Person), `glossary.md`.
