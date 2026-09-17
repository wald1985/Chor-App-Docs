# 0009: Repertoire — folders, attached library books and themes in a Community

**Status:** Accepted (2026-09-17; revised the same day twice: printed editions are live-linked from
the library and read-only; Mappe is a Community `Folder` entity, several per Community; themes come
from the library and from the Community)
**Scope:** chor-app-server (`RepertoireModule`), chor-app-client, `domain-model.md`, `glossary.md`
**Related:** ADR 0003, ADR 0007, ADR 0008, ADR 0010 (library, live link, numbering, library themes),
`brainstorm/repertoire-books-and-legacy-import.md`, `chor-app-server/docs/feature/repertoire/REPERTOIRE_DESIGN.md` (draft, predates this ADR)

## TL;DR for agents
- A Community's repertoire = **attached library books** (read-only, live) + **its own folders**.
- **Book** (UI: *Buch*) = printed edition. Lives only in the global library (`LibraryModule`, ADR 0010).
  A Community attaches it (`BookAttachment`); nobody in a Community can change its songs or its library themes.
- **Folder** (UI: *Mappe*) = Community-owned entity, similar to a book, created by users; a Community
  can have **several** (e.g. "Mappe", "Andere", "Weihnachten 2026"). Filled with custom songs,
  fully editable with `REPERTOIRE_MANAGE`. Numbering always per folder. `isNew` only on folder songs.
- **Themes have two sources:** library themes (come with the attached books, live, read-only) and
  **Community custom themes** (`Theme`), which can be attached to **any** song visible in the Community.
- Theme search in a Community covers both sources across folders and attached books.
- The server knows no book or folder names. No `BookRules`, no `SongCollectionType`, no NewSongs,
  no `BookKind` column — the source (library book vs. Community folder) *is* the kind.
- Other features reference songs by `songId` and resolve them **only** via Repertoire's `SongLookup` port.

## Context
`domain-model.md` modelled three fixed collections. The draft `REPERTOIRE_DESIGN.md` used a free-text
`Song.book`, a hardcoded book list, `BookRules` for `isNew` and a one-click default list of 30 themes.
Product decisions on 2026-09-17: printed books are maintained centrally in a global library and
changes there reach every Community; Community users can't change printed editions; Mappe is a
Community entity like a book, several per Community, with custom songs; the library contains sets of
main themes delivered together with the books; Communities can create custom themes and attach them
to any of their songs.

## Decision

### Ownership
| | Book (printed edition) | Folder (*Mappe*) |
|---|---|---|
| Owner / module | global library, `LibraryModule` (ADR 0010) | one Community, `RepertoireModule` |
| In a Community via | `BookAttachment` (live link, no copy) | own `Folder` row |
| Who changes songs | superadmin only | members with `REPERTOIRE_MANAGE` |
| Numbering | per book or continuous across a series (ADR 0010) | per folder |
| `isNew` | — | allowed |
| Themes on its songs | library themes (read-only) + Community custom themes | library themes + Community custom themes |

### Model in `RepertoireModule`
```
Folder                                (aggregate root, UI: Mappe)
  id, communityId
  title, titleKey                     unique per Community (case-insensitive, incl. archived)
  archivedAt?

FolderSong                            (aggregate root)
  id, communityId, folderId           folderId and number immutable after creation
  number, numberKey, sortKey          string ("22", "22a"); unique (folderId, numberKey) incl. archived
  title, author?, arranger?, isNew
  archivedAt?

BookAttachment
  id, communityId, libraryBookId      no FK into library tables (ADR 0008); validated via LibraryModule read port
  archivedAt?                         detach = archive; unique (communityId, libraryBookId)

Theme                                 (aggregate root, Community custom theme)
  id, communityId, name, nameKey      unique per Community; create / rename / archive / restore
  archivedAt?

SongThemeAssignment                   (Community data; never changes an edition)
  communityId
  songId                              a FolderSong or a LibrarySong visible in this Community
  themeSource: COMMUNITY | LIBRARY    LIBRARY themes may be assigned to folder songs
  themeId                             Theme.id or LibraryTheme.id
```

### Rules
| Rule | Enforced in |
|---|---|
| No Community use case creates/changes/archives library books, library songs or their library theme links | no such use cases; attempts → `PrintedEditionReadOnlyError` (403) |
| `isNew` only on folder songs | `FolderSong` aggregate |
| A new folder song's folder exists, is in the same Community, is active | use case via `FolderRepository` |
| Attach only existing, active library books | use case via `LibraryModule` read port |
| A custom theme can be assigned to any song visible in the Community (folder song or song of an attached book) | use case; song resolved via `SongLookup` |
| Folders, folder songs, themes, attachments are archived, never deleted | aggregates |
| All Community writes gated by `REPERTOIRE_MANAGE` (ADR 0007) | controllers |

### Theme decisions (2026-09-17)
- Library themes **may be assigned to folder songs** (as a Community `SongThemeAssignment` with `themeSource = LIBRARY`).
- A Community custom theme **may have the same name** as a library theme. The theme filter shows
  such themes as **one entry per normalized name** (`nameKey`) and matches songs of both.
- Further theme topics (theme sets, hiding library themes) are postponed.

### Reading and searching
- Song list = folder songs + songs of attached library books (live, via `LibraryModule` read port),
  merged in the application layer (≈2000 songs per Community; no cross-module SQL).
- A song's effective themes = its library themes (library songs only) + Community `SongThemeAssignment`s.
- Visible themes in a Community = library themes used by attached books + Community custom themes.
  The theme filter is by normalized theme name and matches library and custom themes of that name.
- `SongLookup` (exported) returns one shape:
  `{ songId, source: LIBRARY | COMMUNITY, container: { id, title, type: BOOK | FOLDER, seriesId? }, number, title, author, arranger, isNew, themes[{ source, id, name }] }`.

## Consequences
- "Every song is editable" applies to folder songs only.
- Draft design parts replaced: single `songs` table, `book` text, `BookRules`, catalog copy (FR-13),
  default 30 themes (FR-11 — the main themes now come from the library, ADR 0010).
- Logs will hold `songId` without a DB FK for library songs; integrity relies on archive-only in both modules.
- `REPERTOIRE_DESIGN.md` §2 (FR-1…FR-13), §6–§9, D1/D3/D4/D7/D8, OQ-1/OQ-5 must be revised.
  `glossary.md`: Book (Buch), Folder (Mappe — now an entity, several per Community), BookAttachment,
  Theme (custom) vs. LibraryTheme, `isNew`; remove `SongCollectionType`, NewSongs.

## Open questions
| # | Question | Proposal |
|---|---|---|
| Q3 | Can a Community hide library themes it doesn't use? | not in MVP |
| Q4 | Display order of folders and attached books (`sortOrder` per Community) or by title? | open |
| Q5 | Archiving a folder with active songs | allowed; songs stay, no new songs |
| Q6 | Detaching a library book referenced by logs | allowed (archive the attachment); logged songs stay resolvable |
| Q7 | Is a first folder created automatically with the Community? | open |
| Q8 | Prototype Mappe (115 songs) → Community folder via Legacy Import (#9) only? | yes |

## Alternatives considered
- **Free-text book / hardcoded list / server enum** — no validation, migrations per book.
- **`Book` with `kind` for both printed books and Mappe** — they differ in owner, write rights, numbering and `isNew`; separate entities are clearer.
- **Copy printed editions into Communities** — corrections wouldn't propagate; Communities could alter editions.
- **One `songs` table with nullable `communityId`** — one module writing another module's rows (ADR 0008).
- **Copy library themes into Community themes** — library renames/additions wouldn't propagate.
