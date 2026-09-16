# Brainstorm: Repertoire — books, numbering, legacy import

**Date:** 2026-09-16
**Status:** open brainstorm. Not a decision record, not a design. It exists
because `chor-app-server/docs/feature/repertoire/REPERTOIRE_DESIGN.md` cannot
be finalized until the open questions below are answered.
**Related:** `chor-app-server/docs/feature/repertoire/REPERTOIRE_RESEARCH.md`
(facts), `REPERTOIRE_DESIGN.md` (draft; its book-related parts are superseded
by this brainstorm), `capability-breakdown.md` (#5 Repertoire, #9 Legacy Import).

---

## 1. Decisions already made by the product owner

These are settled and carry into the design once it is unblocked.

- **Empty Community.** A new Community starts without songs; the prototype
  catalog is loaded on request.
- **Everything editable.** Any song can be edited; removal = archive. Book
  and number of a song never change after creation.
- **Song number is a string** (`22`, `22a`, `22b`), especially in Mappe.
  Needs a normalized uniqueness key and a natural sort key
  (`22 < 22a < 22b < 100`).
- **`Theme` is the entity name** (UI: *Thema*). Each Community has its own
  editable theme list; a default list (the prototype's 30 themes) can be
  created in one click. A song has a list of themes; custom themes can be
  attached to songs of any book; theme search works across all books.
- **"Neue Lieder" is not a collection but a flag on the song** (`isNew`) —
  a song stops being "new" after some time. A new song must belong to
  **Mappe** or **Andere**.
- **A `Book` entity will be added** (replaces "book is a free-text field on
  the song").
- **A book is a fixed printed edition** — it never contains new songs.
  Therefore `Book.allowsNewSongs` was **rejected**. New songs live in Mappe
  or Andere (which may exist on paper, but are not printed editions).
- Prototype book-section names (`thema_buch_original`) and theme sources
  (`quellen`) are **not** migrated.

## 2. What the prototype actually does (verified 2026-09-16)

Verified by loading `chor-app_v3.html` in a headless browser.

- The "Sammlung" dropdown offers one entry **"Bücher 1-4 (fortlaufend
  nummeriert)"**, plus Mappe and Neue Lieder. Buch 2/3/4 cannot be chosen
  explicitly.
- The prototype data assumes **continuous numbering across the four books**:
  Buch 1 = 1–163, Buch 2 = 164–357, Buch 3 = 358–563, Buch 4 = 564–727. The
  book is **derived from the number**: entering 200 shows "Buch 2 · Laut
  rühmet Jesu Herrlichkeit!", 400 → Buch 3, 613/700 → Buch 4.
- Consequence observed by the product owner (screenshot, 2026-09-16): the log
  only ever showed "Buch 1" entries, and a song like "Buch 2, Nr. 14" cannot
  be recorded. Whether this is a **prototype bug** or a **misreading of the
  UI** depends entirely on how the real printed books are numbered (open
  question Q1).
- **Actual input bug:** the number field is `<input type="number">`, but
  Safari lets arbitrary text be typed. `"1-34"` is parsed with `parseInt` as
  `1`, so the wrong song is recorded without any error.

## 3. Open questions (block the design)

| # | Question | Why it matters |
|---|---|---|
| **Q1** | **How are the real printed books numbered?** Does Buch 2 start at 164 (continuous, as in the prototype) or at 1 (each book numbered on its own)? | Decides the book model (section 4): uniqueness of a song number, how a song is looked up when logging a performance, and whether the prototype catalog data is correct at all |
| Q2 | Are books reference data maintained by developers (seeded/imported), or can an Administrator add books via the UI? | Endpoints and permissions for books |
| Q3 | Is "Andere" one collection or can there be several "other" collections? | Seed data; whether `COLLECTION` books are user-creatable |
| Q4 | Where do the prototype's "Neue Lieder" (custom songs) go on import — Mappe, Andere, or chosen per song? | Legacy import mapping |
| Q5 | Number format: only digits + letter suffix (`22a`)? Any `22-1`, roman numerals, umlauts? Is `22A` the same as `22a`? | `SongNumber` validation, sort key |
| Q6 | Author / arranger — free text or a reference list? (Not `Person`: that is pianists/conductors.) | `Song` model |

## 4. Candidate book models

Common to all candidates:

```
Book
  id, communityId
  title, titleKey            "Buch 1" / "Mappe" / "Andere"
  kind                       PRINTED_EDITION | COLLECTION
  sortOrder
  archivedAt?

Song
  bookId -> Book
  number, numberKey, sortKey
  title, author?, arranger?, isNew, themes[], archivedAt?
  rule: isNew = true only if book.kind = COLLECTION
```

`kind` describes *what the book is*; the `isNew` rule is derived from it, so
the server does not need to know book names (this replaces the hardcoded
`BookRules` from the draft design).

### Model A — one `Book` per printed volume, numbering per book

Fits if **Q1 = each book starts at 1**.

- Buch 1, Buch 2, Buch 3, Buch 4 are four `PRINTED_EDITION` books; Mappe and
  Andere are `COLLECTION` books.
- Uniqueness: `(bookId, numberKey)`.
- Logging a song: choose the book, then the number.
- Simplest model; matches paper 1:1.
- **Consequence:** the prototype catalog's numbers for Buch 2–4 (164–727)
  would be wrong and must be renumbered or re-entered before import.

### Model B — book series with continuous numbering

Fits if **Q1 = continuous numbering** (prototype is correct).

```
BookSeries? { id, title "Bücher" }       // optional grouping
Book        { ..., seriesId?, volume?, firstNumber?, lastNumber? }
uniqueness: (seriesId ?? bookId, numberKey)
```

- Buch 1–4 are four books in one series; the number is unique across the
  series, so a song can be logged by "Bücher + number" and the volume is
  found automatically (current prototype UX), or by picking the volume.
- Keeps the prototype catalog as-is.
- More complex: a scope key for uniqueness (`seriesId` or `bookId`), ranges
  per volume, and rules for songs whose number is outside every range.

### Rejected along the way

| Option | Why rejected |
|---|---|
| Book as a free-text field on the song | typos split books; no place for book attributes; product owner chose an entity |
| `Book.allowsNewSongs` | a printed edition never gets new songs; the rule belongs to the book kind |
| `Volume` as a child entity of a single "Bücher" book with `numbering: CONTINUOUS \| PER_VOLUME` | covers both cases but is the most complex; Model A or B is enough once Q1 is answered |
| Hardcoded book enum on the server | every new book would need a migration |

## 5. Legacy import ("combine")

**Feasible, data quality is good.** Two different prototype JSON sources:

| Source | Contents | How to obtain |
|---|---|---|
| Embedded catalog (`<script id="song-data">` in `chor-app_v3.html`) | 727 Books songs, 115 Mappe songs, 30 merged themes | extract from the HTML (not downloadable via the UI) |
| Backup export (`Daten & Backup → Export`, `chorlieder-verlauf-backup-YYYY-MM-DD.json`) | `custom_songs`, `extra_themen`, `personen`, `abwesenheiten`, `proben`, `log` | download from the running prototype |

The backup does **not** contain the embedded catalog, so the importer reads
both.

| Data | Effort | Notes | Needs feature |
|---|---|---|---|
| Books | low | depends on Q1 (model A or B) | Repertoire |
| Embedded songs + themes | low | clean data: no gaps, no duplicates, all song themes exist; Model A needs renumbering of Buch 2–4 | Repertoire |
| `custom_songs` | low | number int → string; "Neue Lieder" → Mappe/Andere (Q4); number conflicts → report | Repertoire |
| `extra_themen` | low | theme string → `Theme` by name (create if missing); `(sammlung, nummer)` → `songId` | Repertoire |
| `personen` | low | same name in both lists → one `Person` with both roles | People |
| `abwesenheiten` | medium | `Rolle::Name` → `personId` | Absences |
| `proben` + `log` | medium | log rows hold strings: collection + number → `songId`, names → `personId`; "(nicht in Datenbank gefunden)" rows and names of removed people need a policy (archived placeholder or report) | Rehearsal / Performance Log |

Rough estimate: books + catalog + custom songs + extra themes ≈ 1–2 days
including tests. The rest follows as the corresponding features exist.

**Shape discussed:**

- **CLI script in `chor-app-server`, not an API endpoint.** Run inside the
  container on the host, e.g. `docker compose run --rm server node
  dist/cli/import-legacy.js --community <id> --catalog chor-app_v3.html
  --backup backup.json`. Developers run the import for this choir; no large
  uploads through the API, no timeouts, no extra attack surface. An endpoint
  can be added later if Administrators should self-serve. This would replace
  FR-13 (catalog import endpoint) of the draft design.
- **Goes through the domain**, not raw SQL: Nest application context
  (`NestFactory.createApplicationContext`) calling the same use cases /
  repositories, so number normalization, uniqueness, sort keys and theme
  links behave exactly like the API.
- **`--dry-run` by default:** read, map, print a report (to create / skipped
  and why: number conflicts, unresolved songs, unknown names). Writes only
  with `--apply`.
- **Idempotent:** matching on natural keys (book + number, theme name, person
  name); existing data is never overwritten.
- **Transactions:** catalog and songs in one transaction; later log import in
  a separate one.
- **Modular:** one importer per backup section, added when its feature exists.
- JSON report output kept for traceability.
- Belongs to capability **#9 Legacy Import** in `capability-breakdown.md`.

## 6. Next steps

1. Answer **Q1** (real numbering), then Q2–Q6.
2. Pick Model A or B; update `REPERTOIRE_RESEARCH.md` with the verified
   prototype behavior (section 2) and the input bug.
3. Rewrite the book-related parts of `REPERTOIRE_DESIGN.md` (domain model,
   data model, API, catalog section) and replace FR-13 with the CLI import or
   keep both.
4. Decide whether Legacy Import gets its own feature folder
   (`chor-app-server/docs/feature/legacy-import/`) with research → design →
   plan.
5. Update `domain-model.md` and `glossary.md` (Book, `isNew`, removal of
   `SongCollectionType` / NewSongs) once the design is accepted.
