CHOR-APP - README (Legacy Prototype)
=====================================
Stand: September 2026
(c) Daniel Kröcker


WAS IST DAS?
------------
Die Chor-App ist ein einzelnes, eigenständiges HTML-Dokument (chor-app.html),
das im Browser läuft (Chrome oder Edge empfohlen). Es braucht keine
Installation, keinen Server und keine Internetverbindung - Doppelklick auf
die Datei genügt. Daneben gibt es eine inhaltlich gleichwertige Excel-Version
(Chor-App.xlsx) für alle, die lieber in Excel arbeiten.

Verwaltet werden drei unabhängige Liedersammlungen, der Probenverlauf
(Vortrag/Chorprobe) inkl. gruppierter Chorproben mit Einsingen, sowie freie
Themenzuordnungen zu jedem Lied.


DATEIEN IN DIESEM PAKET
------------------------
- chor-app.html          Die App selbst. Einfach im Browser öffnen.
- Chor-App.xlsx           Excel-Version mit denselben Funktionen (Formeln,
                          Dropdowns, Auswertungen).
- Anleitung-Chor-App-Verknuepfung.docx
                          Schritt-für-Schritt-Anleitung, wie man die
                          gemeinsame Datendatei über OneDrive mit der App
                          verknüpft (für die Nutzung durch mehrere Personen).
- README.txt              Dieses Dokument.


DIE DREI LIEDERSAMMLUNGEN
--------------------------
1. Buecher       Bücher 1-4, fortlaufend nummeriert 1-727, mit Themen
                  (bücherübergreifend zusammengeführt).
2. Mappe          Eigene, unabhängige Nummerierung, 115 Lieder als
                  Grunddaten, ohne feste Themenzuordnung.
3. Neue Lieder   Vollständig eigene, freie Sammlung - komplett von dir
                  selbst angelegt und nummeriert, unabhängig von den
                  beiden anderen Sammlungen.

Alle drei Sammlungen lassen sich in "Lieder verwalten" um weitere,
selbst eingetragene Lieder ergänzen.


HAUPTFUNKTIONEN DER APP (REITER)
---------------------------------

Vortrag / Chorprobe (Eintragen)
   Trage ein, welches Lied wann gesungen wurde: Datum, Sammlung, Nummer,
   Klavierspieler, Dirigent, Notiz. Titel und Thema werden automatisch
   nachgeschlagen.

   Chorprobe ist als Sitzung organisiert: Du legst zuerst eine "Probe" mit
   Datum an, wählst das "Einsingen" (zur Auswahl stehen die bekannten
   Dirigenten Paul, Alex, Daniel) und kannst optional eine Notiz
   hinterlegen. Danach trägst du die gesungenen Lieder dieser Probe
   nacheinander ein. In "Bisherige Proben" siehst du alle Proben gruppiert
   mit ihren jeweiligen Liedern, durchsuchbar nach Titel, Nummer,
   Klavierspieler, Dirigent oder Einsingen.

Themensuche
   Durchsucht die Bücher-Sammlung nach Thema und/oder per Suchfeld nach
   Titel ODER Liednummer. Mappe und Neue Lieder haben eigene, separate
   Suchfelder (ohne Themenfilter, da sie bewusst getrennt von der
   bücherübergreifenden Themensuche gehalten werden).

Lieder verwalten
   Hier fügst du neue Lieder zu einer der drei Sammlungen hinzu.
   Zusätzlich kannst du hier jedem bereits bestehenden Lied (aus allen drei
   Sammlungen) über ein Dropdown ein weiteres Thema zuordnen - entweder aus
   der Liste bestehender Themen oder als frei eingetippter, ganz neuer
   Themenname. Neue Themen werden danach überall in der Themensuche
   automatisch mit angeboten (gekennzeichnet als "eigenes Thema").

Auswertung
   Zeigt, wie oft jedes Lied insgesamt, im Vortrag und in der Chorprobe
   gesungen wurde, sowie separate Auswertungen für Klavierspieler und
   Dirigenten.

Daten & Backup
   - Export/Import als Backup-Datei.
   - Lokale Datei verknüpfen: Verbindet die App dauerhaft mit einer Datei
     auf der Festplatte (bzw. in einem geteilten OneDrive-Ordner), sodass
     mehrere Personen dieselben Daten gemeinsam nutzen können. Vor jedem
     Speichern und automatisch alle 20 Sekunden wird die Datei eingelesen
     und mit dem eigenen Stand zusammengeführt (additiv, siehe Hinweis
     unten). Details dazu in der separaten Anleitung
     "Anleitung-Chor-App-Verknuepfung.docx".


WO WERDEN DIE DATEN GESPEICHERT?
----------------------------------
Standardmäßig: im localStorage des Browsers, in dem die App geöffnet ist -
rein lokal auf diesem Gerät, nicht automatisch mit anderen geteilt.

Optional (empfohlen für mehrere Nutzer): über "Daten & Backup" mit einer
Datei auf der Festplatte verknüpfen (z. B. in einem geteilten,
OneDrive-synchronisierten Ordner). Ab dann schreibt und liest die App diese
Datei automatisch und gleicht die Einträge aller Personen zusammen.

Wichtige Einschränkung: Der automatische Abgleich ist ADDITIV - er führt
neue Einträge zusammen, kennt aber keine Löschungen. Gelöschte Einträge auf
einem Gerät können durch ein noch nicht abgeglichenes anderes Gerät
zurückkommen. Es handelt sich also nicht um eine echte Mehrbenutzer-
Datenbank mit Konfliktauflösung, sondern um einen pragmatischen Abgleich,
der für die gemeinsame Nutzung in einem Chor gut funktioniert.


TECHNISCHER HINTERGRUND (FÜR INTERESSIERTE)
---------------------------------------------
- chor-app.html ist eine einzelne, in sich geschlossene Datei: HTML, CSS
  und JavaScript in einer Datei, die Liederdatenbank als eingebettetes
  JSON. Keine externen Abhängigkeiten, keine Serveranbindung nötig.
- Persistenz im Browser über die File System Access API (Chrome/Edge) für
  die optionale Datei-Verknüpfung; localStorage als Grundspeicher.
- Chor-App.xlsx wurde mit denselben Grunddaten erzeugt (openpyxl), mit
  Formeln (INDEX/MATCH, COUNTIFS) statt fester Werte, damit sich die
  Auswertungen automatisch aktualisieren.
- Farbcode in der Excel-Datei: blaue Schrift = Eingabezellen, schwarze
  Schrift = Formeln/generierte Daten (nicht überschreiben).


KONTAKT / URHEBER
------------------
(c) Daniel Kröcker - Chor-App
