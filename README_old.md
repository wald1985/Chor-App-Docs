# Chor-App-Docs

Technische Dokumentation zur **Chor-App** – einer Single-Page-Webanwendung zur Verwaltung von Chorproben, Vorträgen (Auftritten) und der zugehörigen Liederdatenbank.

## 1. Überblick

Die Chor-App ist eine reine Client-Anwendung (HTML/CSS/JavaScript, ohne Build-Prozess, ohne externe Abhängigkeiten), die vollständig im Browser läuft. Sie protokolliert, welche Lieder wann bei Vorträgen und Chorproben gesungen wurden, wer dabei Klavier gespielt bzw. dirigiert hat, und wertet diese Daten aus (Ranglisten, Statistiken pro Person).

| | |
|---|---|
| **Datei** | `chor-app_1.html` |
| **Autor** | Daniel Kröcker |
| **Sprache** | Deutsch (UI), reines Vanilla JS |
| **Abhängigkeiten** | Keine (kein Framework, keine externen Skripte/CDN-Links) |
| **Ausführung** | Direkt im Browser öffnen, kein Server/Build nötig |
| **Datenhaltung** | `localStorage` + optional verknüpfte lokale Datei (File System Access API) |

## 2. Projektstruktur

```
Chor-App/
├── Client/     # aktuell leer – vorgesehen für eine künftige Client-Implementierung
├── Server/     # aktuell leer – vorgesehen für eine künftige Server-Implementierung
└── Doku/       # Git-Repository (github.com/wald1985/Chor-App-Docs)
    ├── README.md
    └── chor-app_1.html   # die eigentliche Anwendung (aktueller Prototyp/Referenzversion)
```

`Client/` und `Server/` sind derzeit Platzhalter. Der aktuelle, funktionsfähige Stand liegt vollständig in der einzelnen Datei `chor-app_1.html` im `Doku`-Ordner – sie enthält HTML, CSS und JavaScript sowie die komplette Liederdatenbank in einem einzigen Dokument.

## 3. Verwendung / Installation

Es ist keine Installation nötig:

1. `chor-app_1.html` in einem aktuellen Browser öffnen (Doppelklick oder per „Öffnen mit…").
2. Für die Funktion **„Lokale Datei verknüpfen"** (empfohlen für dauerhafte Speicherung) wird die *File System Access API* benötigt – diese wird aktuell nur von **Chrome und Edge** (Desktop) unterstützt. In anderen Browsern (Firefox, Safari) funktioniert die App auch, aber ohne diese Funktion; die Daten werden dann ausschließlich im `localStorage` des Browsers gehalten.
3. Kein Internetzugang erforderlich – die App funktioniert vollständig offline.

## 4. Funktionsbereiche (Tabs)

Die Navigation oben in der App besteht aus neun Bereichen:

### 4.1 Vortrag
Erfassung eines im Vortrag (z. B. Gottesdienst) gesungenen Lieds: Sammlung (Bücher 1–4 / Mappe / Neue Lieder), Liednummer, Datum, optional Klavierspieler, Dirigent und Notiz. Zeigt zudem die zuletzt eingetragenen Lieder.

### 4.2 Chorprobe
Proben werden als **Termine (Gruppen)** angelegt, denen mehrere Lieder zugeordnet werden können:
- Neue Probe anlegen (Datum, Einsingen/Dirigent, Notiz) – bei bereits existierendem Datum wird die bestehende Probe fortgesetzt statt eine neue zu erzeugen.
- Innerhalb der „aktiven Probe" werden einzelne Lieder mit Klavierspieler, Dirigent und Notiz hinzugefügt.
- Übersicht der meistgeprobten Lieder sowie durchsuchbare Liste aller bisherigen Proben.

### 4.3 Verlauf
Gemeinsame chronologische Historie aus Vortrag- und Chorprobe-Einträgen: wann wurde welches Lied gesungen/geprobt.

### 4.4 Auswertung
Rangliste der meistgesungenen Lieder (Vortrag + Chorprobe zusammen) inklusive vollständiger Liste.

### 4.5 Klavierspieler
Statistik je Klavierspieler: Anzahl gesamt, Anzahl bei Vortrag, Anzahl bei Chorprobe.

### 4.6 Dirigenten
Analog zu Klavierspieler, ausgewertet je Dirigent.

### 4.7 Themensuche
Durchsucht Lieder nach Titel oder Thema – getrennt für: alle 4 Bücher gemeinsam, die Mappe (eigene Nummerierung) und die „Neue Lieder"-Sammlung (eigene Nummerierung).

### 4.8 Lieder verwalten
- **Neues Lied hinzufügen**: eigene Lieder ergänzen (Sammlung, ggf. Buch, Nummer, Titel, optional Thema) – steht danach sofort in Eintragung, Suche und Auswertung zur Verfügung.
- **Bestehendem Lied ein weiteres Thema zuordnen**: zusätzlich zum ursprünglichen Thema (aus Bücher, Mappe oder Neue Lieder) ein oder mehrere weitere Themen zuweisen, per Dropdown oder freier Texteingabe.

### 4.9 Daten & Backup
- **Lokale Datei (empfohlen)**: Verknüpfung mit einer echten Datei auf dem Rechner (File System Access API, Chrome/Edge). Jede Eintragung wird automatisch dauerhaft und offline in diese Datei gespeichert, unabhängig vom Browser-Cache. Optionen: neue Datei anlegen, bestehende Datei verknüpfen, Verknüpfung erneut bestätigen (nach Berechtigungsverlust), Verknüpfung lösen.
- **Backup & Datenverwaltung**: Export des gesamten Verlaufs als JSON, Import eines zuvor exportierten Verlaufs, sowie vollständiges Löschen aller Eintragungen. Ohne verknüpfte lokale Datei ist die Speicherung an den jeweiligen Browser/das jeweilige Gerät gebunden – ein regelmäßiger JSON-Export wird daher empfohlen, insbesondere vor einem Browserwechsel.
- **Datenstand der Liederdatenbank**: Anzeige von Umfang und Vollständigkeit der mitgelieferten Liederdatenbank.

## 5. Datenmodell

### 5.1 Mitgelieferte Liederdatenbank

Die Basisdatenbank ist als JSON in einem `<script id="song-data" type="application/json">`-Tag eingebettet und wird beim Start per `JSON.parse` geladen (`const DB = JSON.parse(...)`).

| Sammlung | Anzahl Lieder | Nummerierung |
|---|---|---|
| Buch 1 | 163 | fortlaufend, buchübergreifend |
| Buch 2 | 194 | fortlaufend, buchübergreifend |
| Buch 3 | 206 | fortlaufend, buchübergreifend |
| Buch 4 | 164 | fortlaufend, buchübergreifend |
| **Bücher gesamt** | **727** | |
| Mappe | 115 | eigene, separate Nummerierung |

Zusätzlich enthält die Datenbank **30 kanonische Themen** (`themen_kanonisch`), die die ursprünglichen Themenzuordnungen der einzelnen Bücher bücherübergreifend zusammenführen.

Jeder Song-Eintrag hat die Form:
```json
{
  "buch": "Buch 1",
  "nummer": 1,
  "titel": "O großer Gott",
  "thema": "Lob und Dank",
  "themen": ["Lob und Dank"],
  "thema_buch_original": "Lob und Dank"
}
```

Laut `meta.hinweis` in der Datenbank sind alle 4 Bücher und die Mappe vollständig und ohne bekannte Lücken oder Dubletten erfasst; neue oder zukünftig entdeckte Lieder können jederzeit über „Lieder verwalten" ergänzt werden.

### 5.2 Nutzerdaten (lokal erzeugt)

Diese Daten werden nicht mitgeliefert, sondern beim Gebrauch der App erzeugt und lokal gespeichert (localStorage-Keys bzw. Felder in der verknüpften Datei):

- **LOG** – die einzelnen Eintragungen (Vortrag- und Chorprobe-Lieder), inkl. Sammlung, Nummer, Datum, Klavierspieler, Dirigent, Notiz; Chorprobe-Einträge referenzieren zusätzlich eine `probeId`.
- **Proben** – die angelegten Chorprobe-Termine (Gruppen), auf die die LOG-Einträge per `probeId` verweisen.
- **Eigene Lieder** – vom Nutzer über „Lieder verwalten" ergänzte Lieder (Sammlung „Neue Lieder" hat keine mitgelieferte Basis und wird komplett vom Nutzer geführt).
- **Zusätzliche Themen** – nachträglich zugeordnete Zusatzthemen zu bestehenden Liedern aus Büchern, Mappe oder Neue Lieder (`{ sammlung, nummer, thema }`).

## 6. Datenspeicherung im Detail

Die App nutzt zwei sich ergänzende Speichermechanismen:

1. **`localStorage`** (immer aktiv): dient als Absicherung und funktioniert in jedem Browser, ist aber an Gerät + Browser gebunden.
2. **Lokale Datei via File System Access API** (optional, Chrome/Edge): Nach Verknüpfung einer Datei werden alle Änderungen automatisch in diese Datei geschrieben (`writeToFile`) – dauerhaft, geräteunabhängig kopierbar und ohne Verlustrisiko beim Leeren des Browser-Caches. Die Verknüpfung kann über eine `IndexedDB`-gestützte Berechtigung (`idbGet`/`idbSet`/`idbOp`) über Sitzungen hinweg wiederhergestellt werden (`reconnectPermission`, `tryReconnectFile`).

Zusätzlich steht ein manueller **JSON-Export/Import** des gesamten Verlaufs zur Verfügung (Funktionen `writeToFile`/Export-Button bzw. `parseIncomingFile`/Import-Button) – empfohlen als regelmäßiges Backup, unabhängig vom gewählten Speichermechanismus.

## 7. Technische Architektur (Code-Überblick)

Die gesamte Logik befindet sich in einem `<script>`-Block am Ende von `chor-app_1.html`. Wichtige Funktionsgruppen:

- **Datenladen/-speichern**: `loadLog`, `saveLog`, `loadProben`, `saveProben`, `loadCustomSongs`, `saveCustomSongs`, `loadExtraThemen`, `saveExtraThemen`, `idbGet`/`idbSet`/`idbOp`
- **Datei-Verknüpfung**: `linkNewFile`, `linkExistingFile`, `reconnectPermission`, `tryReconnectFile`, `unlinkFile`, `writeToFile`, `updateFsStatus`
- **Eintragen**: `addLogEntry`, `doLookupFor`, `renderRecent`
- **Chorprobe**: `aktiveProbe`, `renderAktiveProbe`, `renderAktiveProbeSongs`, `renderProbenUebersicht`, `renderProbeAuswertung`
- **Verlauf/Auswertung**: `renderVerlauf`, `renderAuswertung`, `buildPersonenAuswertung`, `renderKlavierAuswertung`, `renderDirigentenAuswertung`
- **Suche**: `findInBuecher`, `findInMappe`, `findInNeueLieder`, `renderThemensuche`
- **Lieder verwalten**: `renderNeuTable`, `applyExtraThemen`, `findSongForExtraThema`, `populateLiedDropdownFuerThemen`
- **Daten & Backup**: `renderDaten`, `parseIncomingFile`, `uebernehmeGeladeneDaten`
- **Allgemein/UI**: Tab-Umschaltung (`refreshAllViews`, `refreshNachExternerDatenladung`), Theme-Umschaltung Hell/Dunkel, `escapeHtml`, `fmtDate`, `todayStr`

Das UI unterstützt automatisch **Hell-/Dunkel-Modus** über `prefers-color-scheme` sowie einen manuellen Umschalter (`data-theme`-Attribut).

## 8. Bekannte Einschränkungen

- Die dauerhafte Datei-Verknüpfung (File System Access API) funktioniert nur in Chromium-basierten Desktop-Browsern (Chrome, Edge) – nicht in Firefox oder Safari.
- Ohne verknüpfte lokale Datei sind alle Eintragungen an den jeweiligen Browser/das jeweilige Gerät gebunden (`localStorage`); es wird daher empfohlen, regelmäßig einen JSON-Export als Backup zu erstellen.
- `Client/` und `Server/` sind im Projekt aktuell nicht befüllt – eine etwaige künftige Aufteilung in eine echte Client/Server-Architektur ist bislang nicht umgesetzt.

## 9. Repository

- Remote: https://github.com/wald1985/Chor-App-Docs
- Branch: `main`
