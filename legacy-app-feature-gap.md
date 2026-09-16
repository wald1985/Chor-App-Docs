# Chor-App: legacy app feature gap

**Noted:** 2026-09-16

Source: `RELEASE-NOTES-chor-app.txt` — release notes for the legacy single-file
tool (`chor-app_v3.html` / `Chor-App.xlsx`, author Daniel Kröcker) that this
project replaces. The legacy app is still receiving updates in parallel with
the new architecture work — check that file periodically for drift.

The latest legacy release (15.09.2026) and the one before it describe
functionality not yet reflected in `domain-model.md`:

## Absence tracking (Pianist/Conductor)
Legacy tab "Abwesenheiten": absence calendar for Pianist and Conductor
(vacation/time off), "currently absent" flag, full-text search. Not modeled
yet — candidate: a small `Absence` entity (personId, from, to, note) scoped per
Community, or an attribute of `Person`.

## Pianist/Conductor management (CRUD)
Legacy tab "Verwaltung": create/delete Pianist and Conductor records; dependent
dropdowns refresh automatically. Confirms `Person` needs its own management
use cases, not just a reference inside `SungSongEntry`.

## PDF export of history
Legacy "Verlauf" tab: pick a date range (default last 30 days) and generate a
print-ready summary with "Zuletzt gesungen (Vortrag)" — Performance entries,
newest first, with date/collection/song number/title/theme/pianist/conductor/
note — and "Zuletzt geprobt (Chorprobe)" — Rehearsals grouped by date incl.
WarmUp and rehearsal note. Implemented via the browser print dialog (no PDF
library). For the new app this belongs to Reporting (read model over
Repertoire + Rehearsal).

## Backup export/import + OneDrive sync
Legacy app has full backup export/import and OneDrive file sync, including the
absence and pianist/conductor data. Not discussed yet for the new architecture
— flag in case data portability or sync is a real requirement rather than an
artifact of the single-file design.

## Not yet decided
Whether any of the above are real requirements for the rewrite or legacy
implementation artifacts — confirm before folding into `domain-model.md` or an
OpenSpec capability.
