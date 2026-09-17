# Chor-App: capability breakdown

**Status:** proposal (2026-09-16); People and Community Access are in progress as OpenSpec changes.

Proposed split of the app into loosely coupled capabilities. Based on
`domain-model.md` and the legacy app features in `legacy-app-feature-gap.md`.
Each capability is a candidate for its own `/opsx:propose` change and its own
NestJS module (ADR 0002). Coupling rule: capabilities reference each other
**by ID only**, with no shared domain logic.

## Capabilities

| # | Capability | Scope | Depends on |
|---|---|---|---|
| 1 | **Identity & Community** | registration, Community auto-creation, Administrator role, email invites, multi-community membership, active-Community switch | nothing (foundation) |
| 2 | **Notifications** | `EmailSender` port, email delivery | nothing (technical; used by #1) |
| 1a | **Community Access** | `/communities/:communityId/...` guard, hardcoded per-member permissions set by the Administrator (ADR 0007) | #1 |
| 3 | **People** (UI: *Verwaltung*) | `Person` with combinable hardcoded roles (PIANIST/CONDUCTOR), archive instead of delete | #1a (`PEOPLE_MANAGE`) |
| 4 | **Absences** (UI: *Abwesenheiten*) | absence calendar, "currently absent" flag, search | `personId` |
| 5 | **Repertoire** | Folders (*Mappe*, several, custom songs, `isNew`), attachments of library books, custom `Theme`s on any song, `SongLookup` port (ADR 0009) | `communityId`, #10 via read port |
| 6 | **Rehearsal Log** (UI: *Chorprobe*) | rehearsals, WarmUp, sung-song entries with pianist/conductor | `songId`, `personId` |
| 7 | **Performance Log** (UI: *Vortrag*) | standalone performances | `songId`, `personId`; independent of #6 |
| 8 | **Reporting / History** (UI: *Auswertung*, *Verlauf*) | statistics, "not sung for a while", date-range PDF export | read-only over #5–7 |
| 9 | **Legacy Import** | one-off import from the legacy backup / Excel | writes into #3–7; obsolete after migration |
| 10 | **Library** | global library of printed books (`LibrarySeries`, `LibraryBook`, `LibrarySong`, `LibraryTheme`), hybrid numbering, upload CSV/Excel/JSON, live for all Communities; `LibraryAdminModule` with usage warnings (ADR 0010) | #1 (auth); write endpoints guarded by #11 (ADR 0011) |
| 11 | **Superadmin** | separate platform-admin identity (own table, own login, `aud`-typed JWT — ADR 0011), self-management of other superadmins, seed script; guards #10's write endpoints. Server implemented (2026-09-17); admin panel client is a separate future iteration | #1 (shares `JWT_SECRET`/JWT machinery only) |

## Open points
- **Rehearsal + Performance as one or two capabilities.** They share
  `SungSongEntry` but are already separate aggregates. Separate = looser
  coupling; combined ("Singing Log") = less duplication.
- **Absences split from People** so People can ship first and Absences can be
  added later without breaking anything.
- **Reporting is the most loosely coupled** (read-only projection), so it can
  be built last and changed freely.
- **Backup/OneDrive sync is probably not a capability** in the new app: server
  + Postgres makes backups an infrastructure concern, and the transition is
  covered by the one-off Legacy Import.

## Suggested implementation order
1+2 (done) → 1a → 3 and 5 (in parallel) → 6, 7 and 4 → 8 → 9 (once there is a target to
import into).
