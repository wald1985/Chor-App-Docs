# Brainstorm: Attendix — Wettbewerbsanalyse (Anwesenheit & Terminplanung)

**Date:** 2026-09-18
**Status:** offener Brainstorm. Keine Decision, kein Design. Basiert auf vier
Screenshots von `https://www.attendix.de` (Personen-Ansicht, Termine-Ansicht,
Einstellungen/Mehr-Menü, Version 4.8.10) — keine eigene Nutzung der App, keine
Dokumentation gelesen. Alle Aussagen unten sind Interpretation der UI, nicht
verifiziertes Verhalten.
**Related:** `capability-breakdown.md` (#3 People, #4 Absences, #5 Repertoire,
#6 Rehearsal Log, #8 Reporting), `domain-model.md`, `legacy-app-feature-gap.md`
(Abwesenheitstracking Pianist/Dirigent — verwandtes, aber engeres Thema).

---

## 1. Was Attendix ist

Eine SaaS-App für Anwesenheits- und Terminverwaltung von Chören/Gemeinden
("Instanzen", im Screenshot "Gemeinde Kierspe"). Bottom-Navigation mit drei
Bereichen: **Personen**, **Termine**, **Mehr**. Deckt damit einen breiteren,
aber auch anders geschnittenen Funktionsbereich ab als Chor-App: weniger
Fokus auf Repertoire/Liednummern, dafür ein durchgängiges
Anwesenheits-Konzept, das bei uns aktuell nirgends im Domain-Modell steht.

## 2. Screen-für-Screen-Beobachtungen

### 2.1 Personen (`/tabs/player`)
- Liste ist nach **Gruppen** organisiert (hier: "Gemeindechor (1)"), nicht
  flach — eine Person gehört zu (mindestens) einer Gruppe.
- Jede Person zeigt einen **Prozentwert** (hier 0 %) direkt in der Liste —
  vermutlich eine rollierende Anwesenheitsquote über einen Zeitraum.
- Suchfeld + Filter-Icon oben, "+"-FAB unten rechts zum Anlegen einer Person.

### 2.2 Termine (`/tabs/attendance`, URL-Segment heißt bewusst "attendance")
- Liste von Terminen, gruppiert unter "Aktuelle", mit Datum
  ("Do., 24.09.2026") und ebenfalls einem **Prozentwert** (hier 100 %) —
  vermutlich die tatsächliche Anwesenheitsquote *dieses* Termins.
- Die URL selbst heißt `attendance`, nicht `events`/`termine` — bestätigt,
  dass Anwesenheit das zentrale Datenmodell hinter "Termin" ist, nicht der
  Termin als Ereignis an sich.

### 2.3 Mehr / Einstellungen (`/tabs/settings`)
Gegliedert in sechs Abschnitte:

| Abschnitt | Einträge | Interpretation |
|---|---|---|
| **Instanz-Verwaltung** | Einstellungen, Gruppen, Besprechungen, Rollen & Berechtigungen, Dashboard konfigurieren | Verwaltung der Instanz selbst; "Gruppen" als eigene Verwaltungsseite bestätigt 2.1; "Besprechungen" klingt nach einem zweiten Termin-/Ereignistyp neben Proben (z. B. Vorstandssitzungen); "Dashboard konfigurieren" deutet auf ein anpassbares Start-Dashboard |
| **Benutzer & Rollen** | Administratoren (1), Beobachter (0) | Zwei Rollen mit Zähler direkt sichtbar — **Beobachter** ist eine reine Lesezugriff-Rolle, die es bei uns nicht gibt |
| **Werke & Proben** | Werke, Werkhistorie | "Werke" = Repertoire/Stücke; "Werkhistorie" = wann welches Werk zuletzt behandelt wurde — deckungsgleich mit unserem geplanten Reporting (#8) |
| **Planung** | Ablaufpläne, Ad-Hoc Ablaufplanung | Vorlagen für den Ablauf eines Termins (Reihenfolge der Programmpunkte) plus spontane Tagesplanung — bei uns nirgends abgebildet |
| **Organisation** | Organisation ("Nicht verknüpft") | Instanz kann optional einer übergeordneten Organisation zugeordnet werden — Mehrfach-Instanz/Dachverband-Konzept |
| **Auswertung & Daten** | Kalender abonnieren, Statistiken, Export, Archiv | ICS-Kalenderfeed, Statistik-Seite, Datenexport, Archiv-Ansicht |
| **System** | Instanz wechseln, Benachrichtigungen, Passwort ändern, Hilfe & Feedback, Was ist neu (4.8.10), Ausloggen | Reine Account-/App-Funktionen, "Instanz wechseln" = unser "aktive Community wechseln" |

## 3. Abgleich mit Chor-App

### 3.1 Deckt sich mit Bestehendem/Geplantem

- Personen-Verwaltung → **#3 People**.
- Werke / Werkhistorie → **#5 Repertoire** / **#8 Reporting** ("zuletzt
  gesungen").
- Rollen & Berechtigungen, Administratoren → **#1a Community Access**
  (ADR 0007).
- Instanz wechseln → bereits gelöst (aktive-Community-Switch, ADR 0003).
- Archiv → passt zu unserem bestehenden `archivedAt`-Muster.

### 3.2 Neu — nicht im aktuellen Domain-Modell

1. **Anwesenheit als eigenes Konzept (Person × Termin).**
   Unser `Rehearsal`-Entity kennt nur den WarmUp-Leiter und die gesungenen
   Lieder (`SungSongEntry`), aber keine Teilnehmerliste der ganzen
   Chormitglieder. Attendix trackt off­enbar für **jede Person** je Termin
   einen Status (anwesend/entschuldigt/unentschuldigt o. ä.), aus dem sich
   sowohl der Personen-Prozentwert (2.1) als auch der Termin-Prozentwert
   (2.2) ableiten. Das ist eine echte Erweiterung, kein Sonderfall von
   `Person` — betrifft potenziell **#4 Absences**, die aktuell laut
   `legacy-app-feature-gap.md` nur Pianist/Dirigent-Abwesenheiten (Urlaub)
   meint, nicht generelle Chor-Anwesenheit.
2. **Beobachter-Rolle** — dritte Berechtigungsstufe (reiner Lesezugriff)
   neben Administrator/normalem Mitglied. Würde sich in unser bestehendes
   `CommunityPermission`-Modell einfügen, ist aber aktuell nicht vorgesehen.
3. **Gruppen** (Untergliederung von Personen innerhalb einer Instanz). Wir
   haben laut ADR 0003 eine `MusicalGroup`-Ebene bewusst zurückgestellt
   ("eine Community = ein Chor"). Attendix' "Gruppen" wirken leichtgewichtiger
   als eine zweite Community — eher Stimmgruppen/Teilchöre innerhalb einer
   Instanz. Kein Grund, ADR 0003 zu revidieren, aber ein Datenpunkt dafür,
   dass eine leichte Gruppierung *unterhalb* von Community gefragt sein
   könnte.
4. **Ablaufpläne / Ad-Hoc Ablaufplanung.** Vorlagen bzw. spontane Planung des
   Ablaufs eines Termins (Reihenfolge von Programmpunkten), losgelöst von der
   reinen Liederliste. Keine Entsprechung bei uns; wäre am ehesten eine
   Erweiterung von **#6 Rehearsal Log** oder eine eigene kleine Capability.
5. **Besprechungen** als zweiter Termintyp neben Proben — offen, ob das für
   uns relevant ist (Vorstandssitzungen o. ä. sind kein Chor-Anliegen, aber
   evtl. ein allgemeineres "Termin"-Konzept dahinter).
6. **Kalender-Abo (ICS-Feed)** der Termine — sinnvolle Ergänzung zu unserem
   geplanten Reporting/Export, aktuell nicht geplant.
7. **Organisation/Dachverband** — mehrere Instanzen unter einer Organisation.
   Bei uns bisher kein Thema; nur relevant, falls ein Verband mehrere
   Gemeinden/Chöre zentral verwalten will.
8. **Konfigurierbares Dashboard** — reine UX-Ergänzung, keine Domain-Auswirkung.

### 3.3 Bewusst nicht übernehmen (Begründung)

- **Zweite Community-Ebene für "Organisation"** ist aktuell außerhalb des
  MVP-Scopes (ADR 0003 bleibt bestehen), bis ein konkreter Bedarf
  (Dachverband mit mehreren Gemeinden) vorliegt.
- **"Besprechungen" als eigener Termintyp** wirkt wie ein
  Gemeindeverwaltungs-Feature, das über den Chor-Fokus von Chor-App
  hinausgeht — vor Übernahme klären, ob das wirklich gebraucht wird.

## 4. Offene Fragen (vor einer Design-Entscheidung zu klären)

| # | Frage | Warum relevant |
|---|---|---|
| Q1 | Ist "Anwesenheit" nur für reguläre Chormitglieder gedacht, oder auch für Pianist/Dirigent (dann Überschneidung mit `legacy-app-feature-gap.md`)? | Entscheidet, ob #4 Absences erweitert oder eine neue Capability "Attendance" entsteht |
| Q2 | Wird Anwesenheit pro Termin manuell erfasst (Haken setzen) oder aus etwas anderem abgeleitet? | API-/UI-Aufwand |
| Q3 | Braucht Chor-App wirklich "Gruppen" unterhalb einer Community, oder reicht die flache Personenliste? | Vermeidet verfrühte Modellierung |
| Q4 | Ist "Ablaufpläne" ein echtes Bedürfnis (Programmreihenfolge) oder reicht die bestehende Liederliste je Rehearsal? | Scope von #6 |
| Q5 | Ist die Beobachter-Rolle für unsere Nutzergruppe (kleine Chöre) überhaupt gefragt? | Vermeidet unnötige Berechtigungskomplexität |

## 5. Nächste Schritte

1. Q1–Q5 mit dem Product Owner klären.
2. Falls Anwesenheit bestätigt wird: `domain-model.md` um eine
   `AttendanceRecord`-artige Entität (personId, rehearsalId/terminId, status)
   ergänzen und gegen **#4 Absences** abgrenzen (ggf. #4 umbenennen/erweitern
   statt eine Parallelstruktur zu bauen).
3. Beobachter-Rolle ggf. als zusätzlicher `CommunityPermission`-Wert bzw.
   eigene Nicht-Mitglieder-Rolle in ADR 0007 nachtragen.
4. Ablaufpläne/Ad-Hoc-Planung erst nach #6 (Rehearsal Log) evaluieren, nicht
   vorher — kein Blocker für den aktuellen Implementierungsplan.
5. Keine Code-Änderungen aus diesem Dokument ableiten, ohne die obigen
   Fragen beantwortet zu haben (nur Brainstorm, keine Decision).
