# Chor-App: capability breakdown

**Status:** Living reference (updated 2026-09-17). Capabilities #1, #2, #1a, #10, and #11 are implemented on both server and client.

Proposed split of the app into loosely coupled capabilities. Based on
`domain-model.md` and the legacy app features in `legacy-app-feature-gap.md`.
Each capability is a candidate for its own `/opsx:propose` change and its own
NestJS module (ADR 0002). Coupling rule: capabilities reference each other
**by ID only**, with no shared domain logic.

## Capabilities

| # | Capability | Scope | Depends on | Implementation Status |
|---|---|---|---|---|
| 1 | **Identity & Community** | registration, Community auto-creation, Administrator role, email invites, multi-community membership, active-Community switch | nothing (foundation) | **Done** (server + client) |
| 2 | **Notifications** | `EmailSender` port, email delivery (SMTP/Nodemailer) | nothing (technical; used by #1) | **Done** (server) |
| 1a | **Community Access** | `/communities/:communityId/...` guard, hardcoded per-member permissions set by the Administrator (ADR 0007) | #1 | **Done** (server + client) |
| 10 | **Library** | global library of printed books (`LibrarySeries`, `LibraryBook`, `LibrarySong`, `LibraryTheme`), hybrid numbering, upload CSV/Excel/JSON, live for all Communities; `LibraryAdminModule` with usage warnings (ADR 0010) | #1 (auth); write endpoints guarded by #11 (ADR 0011) | **Done** (server + client + seed data) |
| 11 | **Superadmin** | separate platform-admin identity (own table, own login, `aud`-typed JWT — ADR 0011), self-management of other superadmins, seed script; guards #10's write endpoints; admin panel in client (`/admin/*`) | #1 (shares `JWT_SECRET`/JWT machinery only) | **Done** (server + client) |
| 3 | **People** (UI: *Verwaltung*) | `Person` with combinable hardcoded roles (PIANIST/CONDUCTOR), archive instead of delete | #1a (`PEOPLE_MANAGE`) | **Spec'd** (`openspec/changes/add-people`) |
| 5 | **Repertoire** | Folders (*Mappe*, several, custom songs, `isNew`), attachments of library books (`BookAttachment`), custom `Theme`s on any song, `SongLookup` port (ADR 0009) | `communityId`, #10 via read port | **Designed** (ADR 0009; client session copy in place) |
| 4 | **Absences** (UI: *Abwesenheiten*) | absence calendar, "currently absent" flag, search | `personId` | Planned |
| 6 | **Rehearsal Log** (UI: *Chorprobe*) | rehearsals, WarmUp, sung-song entries with pianist/conductor | `songId`, `personId` | Planned |
| 7 | **Performance Log** (UI: *Vortrag*) | standalone performances | `songId`, `personId`; independent of #6 | Planned |
| 8 | **Reporting / History** (UI: *Auswertung*, *Verlauf*) | statistics, "not sung for a while", date-range PDF export | read-only over #5–7 | Planned |
| 9 | **Legacy Import** | one-off import from the legacy backup / Excel | writes into #3–7; obsolete after migration | Planned |

## Open points
- **Rehearsal + Performance as one or two capabilities.** They share
  `SungSongEntry` but are already separate aggregates. Separate = looser
  coupling; combined ("Singing Log") = less duplication.
- **Absences split from People** so People can ship first and Absences can be
  added later without breaking anything.
- **Reporting is the most loosely coupled** (read-only projection), so it can
  be built last and changed freely.
- **Backup/OneDrive sync is not a capability** in the new app: server
  + Postgres makes backups an infrastructure concern, and the transition is
  covered by the one-off Legacy Import.

## Implementation progress & order
- **Completed:** 1 (Identity), 2 (Notifications), 1a (Community Access), 10 (Library & 727 Songs seed), 11 (Superadmin), Client Dashboard Shell.
- **Next in line:** 3 (People) and 5 (Repertoire) → 6 (Rehearsal Log) & 7 (Performance Log) & 4 (Absences) → 8 (Reporting) → 9 (Legacy Import).

