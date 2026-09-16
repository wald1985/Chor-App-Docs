# Chor-App — glossary (German UI ↔ English domain/code)

**Policy (decided 2026-09-15):** the target UI/UX is **German** (that's the
audience — a German-speaking choir). The **domain model and all
development is English** — entity/class/field names, API, specs, ADRs,
code comments. This supersedes the project's earlier "keep German terms
in code" instruction from initial setup.

**Exception (decided 2026-09-16):** documentation inside `chor-app-client`
and `chor-app-server` (`docs/`, e.g. `docs/feature/<feature>/`) is written
in **Russian**, using the English domain names from this table for domain
terms. Code and this repo's specs/ADRs stay English.

This file is the single source of truth for the mapping. Don't invent an
alternate English name for a German term, or vice versa, without updating
this file first — specs, `domain-model.md`, and code should all agree with
it.

## Entities / concepts

| German (UI)          | English (domain/code) | Notes |
|-----------------------|------------------------|-------|
| Lied / Lieder         | Song / Songs           | |
| Chorprobe / Probe     | Rehearsal               | one rehearsal session |
| Vortrag                | Performance              | standalone; no Rehearsal required (the "spontan" case) |
| Einsingen               | WarmUp                    | warm-up portion of a Rehearsal; has its own leader |
| Dirigent                 | Conductor                  | a role a Person can hold |
| Klavierspieler            | Pianist                      | a role a Person can hold |
| Thema / Themen             | Theme / Themes                | |
| Bücher (Buecher)             | Books                           | one of the three song-collection types |
| Mappe                          | Folder                            | one of the three song-collection types — decided (not a literal "binder"/"portfolio" translation) |
| Neue Lieder                      | NewSongs                            | one of the three song-collection types |
| Auswertung                         | Report                                | reporting/statistics; no entities of its own |
| Gemeinschaft                         | Community                              | the tenant — decided 2026-09-15; domain/code always says "Community", UI shows "Gemeinschaft" |
| Administrator                          | Administrator                            | same word in both languages |
| Benutzer / Mitglied                      | User                                       | UI label not finalized between "Benutzer" and "Mitglied" — pick when building the Identity & Community UI |
| Person                                     | Person                                       | same word in both languages |
| Rolle (Klavierspieler / Dirigent)          | PersonRole (`PIANIST` / `CONDUCTOR`)         | roles of a Person, combinable, hardcoded set (MVP) |
| archiviert                                 | archived                                     | a Person removed from active use; never hard-deleted |
| Berechtigung / Berechtigungen              | Permission / Permissions (`CommunityPermission`) | granted per member by the Administrator, e.g. `PEOPLE_MANAGE` |
| Mitglieder                                 | Members (`CommunityMembership`)              | members of a Community |

## Feature / page names (not entities — for UI↔dev traceability only)

| German (UI)      | English reference |
|-------------------|--------------------|
| Lieder verwalten   | Manage songs        |
| Themensuche         | Theme search          |
| Personen verwalten  | Manage people (replaces legacy "Klavierspieler/Dirigenten verwalten") |
| Berechtigungen verwalten | Manage member permissions |
| Konto               | Account page — own User data, Community memberships, sign out |

## Maintenance rule

Adding a new domain concept? Add its German UI term and English domain
name here in the same change, before using either in a spec or in code.

## Open follow-up

Localization mechanism itself (hardcoded German strings vs. a proper i18n
layer for future multi-language support) is not decided — see
`chor-app-client/AGENTS.md`.
