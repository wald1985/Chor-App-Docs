# 0010: Public book library — global printed editions and main themes, live-linked into Communities

**Status:** Accepted (2026-09-17; revised the same day: live link instead of copy; hybrid numbering;
library themes; superadmin usage warning)
**Scope:** chor-app-server (`LibraryModule`, `LibraryAdminModule`; `RepertoireModule` attachments), chor-app-client, `capability-breakdown.md`
**Related:** ADR 0003, ADR 0007, ADR 0008, ADR 0009, `REPERTOIRE_DESIGN.md` §9 / FR-11 / FR-13 (superseded)

## TL;DR for agents
- **One global library** of printed books, their songs and **main themes**. Not Community-scoped.
  Own module `LibraryModule` (`src/library`).
- **Live link:** a Community attaches a library book (ADR 0009) and always sees current songs and themes.
  Superadmin changes are visible in every Community immediately. Nothing is copied.
- **Only the superadmin changes the library.** Communities read and attach.
- **Library themes** are curated centrally and linked to library songs; they reach Communities with the
  attached books. Community custom themes are Repertoire's business (ADR 0009).
- **Hybrid numbering:** a book is numbered on its own (usual) or belongs to a series with continuous
  numbering (e.g. Bücher 1–4 = 1–727). Decided per book by whoever fills the data.
- Uploads (CSV / Excel / JSON) are parsed and validated **once**; **re-upload updates songs in place by number**,
  so `songId`s stay stable.
- **Superadmin gets a usage warning** before archiving something Communities use. The warning is composed in
  `LibraryAdminModule`, which imports both `LibraryModule` and `RepertoireModule` — `LibraryModule` itself
  never depends on Repertoire.
- `/admin/library/...` lives in `LibraryAdminModule` behind a placeholder `SuperAdminGuard`. Reads: any authenticated user.
- No publication status, `publishedAt` or editions in the MVP.

## Context
Printed songbooks and their thematic index are the same for every choir. They must be maintained centrally,
corrections must reach every Community, Community users must not alter them. Most books are numbered from 1;
a few are numbered continuously across volumes. Content comes from Excel/CSV/JSON.

## Decision

### 1. Modules
| Module | Owns | Depends on |
|---|---|---|
| `LibraryModule` | series, books, songs, library themes; read port for others | Identity (auth guard) only |
| `RepertoireModule` | folders, attachments, custom themes, assignments, `SongLookup`; exports a usage query | `LibraryModule` read port |
| `LibraryAdminModule` | `/admin/library` controllers, upload, usage warnings; later part of the superadmin capability | `LibraryModule`, `RepertoireModule` exports |

No cycles: Library ← Repertoire ← LibraryAdmin.

### 2. Model (MVP)
```
LibrarySeries                      optional grouping with continuous numbering
  id, title, titleKey, archivedAt?

LibraryBook                        (aggregate root, UI: Buch)
  id, title, titleKey              unique in the library
  seriesId?, volume?               null seriesId = numbered on its own
  archivedAt?, createdAt, updatedAt

LibrarySong
  id                               stable UUID, referenced by Communities
  libraryBookId
  numberScopeId                    = seriesId ?? libraryBookId (set by the aggregate)
  number, numberKey, sortKey       unique (numberScopeId, numberKey) incl. archived
  title, author?, arranger?
  archivedAt?

LibraryTheme                       (main themes, e.g. the prototype's 30)
  id, name, nameKey                unique in the library
  archivedAt?

LibrarySongTheme
  librarySongId, libraryThemeId
```
A book's theme set = the library themes linked to its songs (see Q2 for explicit sets).

### 3. Hybrid numbering
- Without series: number unique within the book; lookup `(libraryBookId, number)`.
- In a series: number unique across all volumes; lookup `(seriesId, number)` returns the song with its volume
  ("Bücher + 200 → Buch 2"), or `(libraryBookId, number)`.
- No per-volume number ranges in the MVP; moving a book into/out of a series only if numbers stay unique.

### 4. Upload and changes
- CSV / Excel / JSON → one parser adapter per format → one in-memory structure. Columns: series?, volume?,
  book, number, title, author?, arranger?, themes (names).
- Theme names in the file are matched to `LibraryTheme` by `nameKey`; unknown names are created.
- Validate once, store in one transaction; on error nothing is stored and an error report is returned.
- **Re-upload updates in place:** match by `(numberScopeId, numberKey)` → update fields and theme links;
  new numbers → create; numbers missing from the file → archive. Never delete, never recreate.
- **Dry run first:** an upload returns a preview (created / updated / archived) including usage of items
  that would be archived; applying requires confirmation.

### 5. Superadmin usage warning
- `RepertoireModule` exports `LibraryUsageQuery`: for library book / song / theme ids → number of Communities
  attaching the book, number of Community theme assignments on the song; later extended with log references
  (Rehearsal/Performance modules contribute their own counts to `LibraryAdminModule`).
- Archiving a library series, book, song or theme that is in use → **409 `LIBRARY_ITEM_IN_USE`** with the usage
  numbers; repeating with `confirm=true` archives. Same data is shown in the upload dry run.
- Archived items stay resolvable for existing references and are shown as archived; they can't be newly attached.

### 6. Authorization
- **Read** (`GET /library/...`): any authenticated user (`JwtAuthGuard`).
- **Write** (`/admin/library/...`): `LibraryAdminModule`, placeholder `SuperAdminGuard` (currently allows every
  request). The future superadmin capability replaces only the guard.

### 7. Prototype catalog
Buch 1–4 → series "Bücher", four volumes (1–163, 164–357, 358–563, 564–727); the prototype's 30 merged themes →
`LibraryTheme`s linked to their songs. `legacy-v3.json` may be the first upload. Prototype Mappe is Community data
(ADR 0009 Q8). FR-11 (default 30 themes) and FR-13 (catalog copy) of the draft design are superseded.

## Consequences
- Capabilities: **#10 Library** (incl. `LibraryAdminModule` for now), **#11 Superadmin** future.
- `chor-app-server/docs/feature/library/` needs research → design → plan; `REPERTOIRE_DESIGN.md` rewritten around
  folders, attachments, two theme sources and live reads.
- **Accepted risk until superadmin exists:** anyone reaching `/admin/library` changes content for all Communities.
- A wrong library change is immediately wrong everywhere; no per-Community override in MVP.
- Library ids are part of the contract with Communities.

## Open questions
| # | Question | Proposal |
|---|---|---|
| Q1 | Names and German UI terms (`LibraryModule`, `LibraryBook`, `LibrarySeries`, `LibraryTheme`, *Bibliothek*) → `glossary.md` | open |
| Q2 | "Theme sets": one global list of main themes, or explicit sets per book/series (themes without songs included)? | global list, book set derived from its songs |
| Q3 | First upload formats and file schema (one book per file or several) | open |
| Q4 | Until superadmin: must `/admin/library` require a valid JWT? Deploy to production or keep disabled by a flag? | open |
| Q5 | Do Communities get notified about library changes? | not in MVP |
| Q6 | Changing a library song's number: edit (keep id) or archive + new song? On upload a changed number looks like a new song | edit via admin UI keeps id; upload treats it as new + archived |

## Alternatives considered
- **Library inside Repertoire** — mixes global and Community-scoped data and a platform role.
- **Copy into Communities** — corrections wouldn't propagate; editions could diverge.
- **Usage check inside `LibraryModule`** — needs Repertoire → circular dependency; composing in `LibraryAdminModule` avoids it.
- **Replace all songs on re-upload** — changes ids, breaks references.
- **Strictly per-book or strictly continuous numbering** — each is wrong for some real books.
