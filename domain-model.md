# Chor-App — domain model (draft)

Working reference for domain entities/value objects, organized by
candidate bounded context (see `openspec/config.yaml` and
`decisions/0002-server-stack.md`). This feeds each capability's future
`design.md` under `openspec/changes/<change-id>/` — it is not itself an
OpenSpec artifact, and not a final word: expect it to be refined once
specs are written per capability. Per ADR 0002, none of this lives with
Prisma types — these are `domain/` entities/VOs, framework- and DB-free.

**Language:** entity/field names here are **English** (domain/code), per
`glossary.md` — the target UI is German, but the domain and all
development are English. Where useful, a name's German UI term is noted
in parentheses on first mention; don't take that as the name to use in
code.

**Tenant scoping (per ADR 0003):** every entity below except `Community`
itself belongs to exactly one `Community`. Treat `communityId` as an
implicit field on every aggregate root in Repertoire, Rehearsal &
performance log, and Reporting — omitted from the per-entity notes below
to avoid repeating it everywhere, not because it's optional.

## Identity & Community (tenancy)

- **Community** (UI: *Gemeinschaft*) — entity, tenant root. Created
  automatically when the first user registers (see ADR 0003) — there's no
  separate "create a community" step. All other domain data is scoped to
  one Community.
- **User** — entity: account. **Can belong to multiple Communities**
  (revised 2026-09-15) via a `CommunityMembership` (userId, communityId,
  role) join, each with its own Role — not a single `communityId` on
  User. Client/session needs an "active Community" selector, like
  switching workspaces in Slack. The user who registers first becomes
  **Administrator** of the newly created Community; an Administrator can
  then invite further Users **by email** (decided — typically conductors,
  choir wardens). Roles beyond Administrator vs. a general member, and
  the exact invite-email mechanics (token/expiry/resend), are not decided
  yet. A User can change their own password, or recover it via an emailed
  single-use code (`identity/password-management`, decisions/0004);
  internally carries a `tokenVersion` counter so a password change/reset
  invalidates other outstanding sessions — an implementation detail, not
  something other capabilities need to know about.
- **No `MusicalGroup`/ensemble layer** — considered and deliberately
  deferred (see ADR 0003): one Community *is* one musical group (choir or
  orchestra); an organization running a second ensemble creates a second
  Community. Multi-community membership (above) means the same person
  doesn't need a second login for it — what's still not shared across
  Communities is `Person` records and Reporting (each Community's stats
  stay separate; expected, not a gap). Revisit `MusicalGroup` only if
  sharing repertoire/stats across ensembles becomes a confirmed real
  need.

## Repertoire (song catalog)

- **Song** (UI: *Lied*) — entity, aggregate root. Identity: `(collection,
  number)` — numbering is independent per collection (Books 1-727
  continuous across 4 books, Folder its own 1-115, NewSongs free/
  user-assigned). Attributes: title, its Themes.
- **SongCollectionType** — value object / enum: `Books | Folder |
  NewSongs` (UI: *Bücher / Mappe / Neue Lieder*). Not an entity — fixed
  set of three, no independent lifecycle, but each has different rules
  (only Books participates in the cross-book theme search).
- **Theme** (UI: *Thema*) — entity, not a bare string: themes are created
  ad hoc and reused everywhere afterward, so they need stable identity to
  avoid near-duplicate spellings drifting apart. Many-to-many with Song.

## Rehearsal & performance log

- **Rehearsal** (UI: *Chorprobe* / *Probe*) — entity, aggregate root:
  date, WarmUp leader (a Person acting as conductor), optional note, and
  its list of `SungSongEntry`. Invariant scoped strictly to *this*
  aggregate: a `SungSongEntry` cannot exist inside a Rehearsal without
  that Rehearsal already existing (you always create it first, then add
  songs to it). This invariant does **not** apply to Performance.
- **Performance** (UI: *Vortrag*) — entity, its own (lighter) aggregate: a
  single standalone performance — date + one `SungSongEntry`. **Fully
  independent of Rehearsal by design** — covers the "spontan" case: a
  song performed directly, with no related rehearsal at all. Never
  requires a Rehearsal.
- **SungSongEntry** — value object, reused inside both Rehearsal (as a
  list item) and Performance: references a Song, a Person as pianist, a
  Person as conductor, plus a note.
- **Person** — entity. Shared identity for both the Pianist and Conductor
  roles (UI: *Klavierspieler*, *Dirigent*) — a Person can hold either or
  both — decided over keeping these as free text, specifically so
  Reporting aggregates correctly and doesn't fragment on spelling
  variants. Known instances today: Paul, Alex, Daniel — but modeled as
  data, not a hardcoded enum, so people can be added/removed later.
  **Separate from `User`** (see ADR 0003): a Person has an *optional*
  link to a User, since some pianists/conductors are guests with no
  login. Not merging the two was a deliberate decision.

## Reporting

- No new entities. This is a read-model/projection computed from
  Repertoire + Rehearsal & performance log data (counts per Song, per
  Person-as-pianist, per Person-as-conductor) — lives at the application
  layer, not as its own persisted domain concept, at least for now. UI
  label: *Auswertung*.

## Notifications

No entities yet — just the `EmailSender` port from
`decisions/0002-server-stack.md`, implemented (`src/notifications/` in
chor-app-server) and in real use for the `identity/password-management`
forgot-password email. An entity like `EmailMessage` would only appear if a
queue/template system is needed later. Also used for the Community invite
flow (ADR 0003, not yet built).

## Relationships (summary)

- `Community` 1..* `User`, 1..* `Song`/`Theme`/`Rehearsal`/`Performance`/`Person`
  (tenant scoping — see note at top)
- `User` → 1 `Community`, has a Role; optionally linked from a `Person`
- `Song` 1..* `Theme` (many-to-many via join)
- `Rehearsal` 1..* `SungSongEntry`
- `SungSongEntry` → 1 `Song`, → 1 `Person` (as pianist), → 1 `Person`
  (as conductor)
- `Performance` → 1 `SungSongEntry` (or embeds song/pianist/conductor/note
  directly — equivalent, decide at design time for that capability); no
  relation to `Rehearsal`
- `Person` can hold one or both roles: conductor, pianist — not mutually
  exclusive; optionally linked to a `User`

## Still open (don't assume — decide when the relevant capability is spec'd)

- Bounded-context boundaries are still informal groupings above, not a
  final decision.
- Aggregate ID strategy (UUID vs. sequence) per entity.
- Whether `Song` uniqueness `(collection, number)` is enforced at the
  domain layer, the DB layer, or both.
- Whether `Performance` should literally reuse the `SungSongEntry` value
  object type or just share its shape.
- Invite email mechanics and role granularity beyond Admin/member (see ADR
  0003). Auth mechanism itself is decided — email+password, JWT bearer
  token, see `decisions/0004-auth-mechanism.md`.
- Localization mechanism for the German UI (hardcoded strings vs. an i18n
  layer) — see `glossary.md`.
