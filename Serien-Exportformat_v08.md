# Serien-Exportformat und Rating-Stand (Stand v57 / v61 / v71 / Serienwertung v11)

Alle drei Turnierpläne schreiben ihrem JSON-Export seit v43 bzw. v49 einen
zusätzlichen Block `serie` bei. Nur dieser Block wird von der Serienwertung
gelesen; die Auswertungsseite rechnet keine Ranglisten nach.

```json
"serie": {
  "formatVersion": 1,
  "modus": "einzelgruppe" | "zwei-gruppen" | "gruppen-ko",
  "disziplin": "9-Ball",
  "datum": "2026-06-07",
  "teilnehmer": 14,
  "vollstaendig": true,
  "ranking": [ { "platz": 1, "name": "Volker" }, ... ]
}
```

- `datum`: Zeitpunkt des ersten eingetragenen Ergebnisses (`tournamentStart`),
  ersatzweise das aktuelle Datum. In der Auswertungsseite je Turnier änderbar.
- `vollstaendig`: false, solange Spiele offen sind bzw. Phase 2 / KO-Runde nicht
  gestartet oder nicht entschieden ist. Solche Turniere werden nur mit
  ausdrücklichem Häkchen eingelesen.
- `ranking`: fertige Endplatzierung. Im KO-Modus können sich mehrere Spieler
  einen Platz teilen (dichte Platzvergabe), z.B. 1,2,3,4,5,5,5,5,9,9,9,9,...
  Geteilte Plätze ergeben in der Serienwertung gleiche Punkte.

Erzeugt wird der Block in jeder Datei durch `buildSerienExport()`, eingebunden
in `getCurrentSnapshot()` innerhalb eines try/catch, damit ein Fehler dort das
Speichern des Turnierstandes nicht verhindert.

## Formatversion 2: Partien für das Vereins-Rating

Ab Einzelgruppe v52, Zwei Gruppen v55 und Gruppen mit KO v65 enthält der Block
`serie` zusätzlich zwei Felder. Die Serienwertung v07 ignoriert beide und liest
Exporte der Formatversion 2 unverändert ein.

```json
"serie": {
  "formatVersion": 2,
  "...": "Felder wie oben",
  "ratingWerten": true,
  "partien": [
    { "a": "Volker", "b": "Olaf", "satzA": 5, "satzB": 3,
      "vorgabeA": 0, "vorgabeB": 0, "raceTo": 5, "phase": "Gruppe A" }
  ]
}
```

- `ratingWerten`: Ankreuzfeld "Für Vereins-Rating werten" in der
  Einstellungsleiste. Gehört zum Turnierstand (Snapshot-Feld `ratingWerten`),
  ältere Stände ohne das Feld gelten als `true`. "Turnierplan zurücksetzen"
  setzt es wieder auf `true`. Die Serienwertung übernimmt den Wert beim
  Einlesen, dort ist er nachträglich änderbar und maßgeblich.
- `partien`: nur beendete Partien (eine Seite hat das Race to der Phase
  erreicht). Freilose und Zwischenstände fehlen. `null` bedeutet, dass die
  Partien wegen eines Programmfehlers nicht ermittelt werden konnten; die
  Endplatzierung wird trotzdem exportiert. Eine leere Liste bedeutet "keine
  beendeten Partien".
- `a`, `b`: Namen wie beim Export im Turnierplan eingetragen.
- `satzA`, `satzB`: Endstand inklusive Vorgabe.
- `vorgabeA`, `vorgabeB`: Vorgabe des Handicaps, im Satzstand enthalten.
  Ohne Handicap immer 0.
- `raceTo`: Race to der Phase, in der die Partie gespielt wurde.
- `phase`: feste Beschriftung ohne Rundennummer, damit eine Partie auch nach
  einem Neuaufbau des Spielplans (Spieler nachtragen) gleich heißt:
  - Einzelgruppe: `Gruppe`
  - Zwei Gruppen: `Gruppe A`, `Gruppe B`, `Platzierung` (Phase 2)
  - Gruppen mit KO: `Gruppe A` bis `Gruppe D`, `Achtelfinale`,
    `Viertelfinale`, `Halbfinale`, `Finale`, `Spiel um Platz 3`,
    `Platzierung` (Phase 3)
- Kennung einer Partie: beide Namen normalisiert und sortiert plus `phase`.
  Zwei Spieler begegnen sich je Phase höchstens einmal.

Erzeugt wird die Liste durch `buildPartienExport()` (je Datei eigene
Umsetzung) über `partienSicher()`, das Fehler abfängt. Gemeinsame Bausteine:
`partieEintrag()` und `partienDerGruppe()`.

Alte Turnierexporte erhalten die Partien, indem sie in die aktuelle
Turnierplan-Version importiert und neu exportiert werden.

## Punkteformel der Serienwertung

`punkte = teilnehmer + 1 - platz`, der Sieger erhält einen Zusatzpunkt.
Entspricht der bisherigen Excel-Formel
`=WENN(V10="";"";V$39+1-V10+WENN(V10=1;1;0))`.

Reihenfolge bei Punktgleichheit: mehr erste Plätze, dann mehr zweite Plätze
usw., zuletzt der Name. Punktgleiche Spieler teilen sich einen Platz.

Streichergebnisse sind je Serie einstellbar ("beste X von N"); gestrichen
werden die punktschwächsten Turniere eines Spielers.

## Browser: Chrome oder Edge, nicht Firefox

Am 25.08.2026 auf dem Notebook geprüft: Firefox behält den localStorage einer
per `file://` geöffneten Datei nicht über das Schließen hinaus, Edge tut es.
Damit sind in Firefox die automatische Sicherung und die Restore-Punkte aller
drei Turnierpläne wirkungslos. Ab v44 / v50 zeigen die Turnierpläne deshalb
einen roten Warnkasten, sobald sie in Firefox laufen (Erkennung über
`navigator.userAgent`).

## Speicherdatei der Serienwertung

Ab v02 kann die Serienwertung ihren Datenbestand in eine echte Datei schreiben
(File System Access API, `showSaveFilePicker` / `showOpenFilePicker`, Dateiverweis
in IndexedDB unter `billard-serienwertung/handles/speicherdatei`). Einmal
verknüpfen, danach schreibt die Seite jede Änderung selbst hinein; beim nächsten
Öffnen ist eine Nutzeraktion nötig (Banner mit Schaltfläche "Zugriff erlauben und
laden"), das verlangt der Browser.

Firefox kennt diese Schnittstelle nicht. Dort erscheint stattdessen ein Hinweis,
und es bleibt bei Export und Import von Hand.

Übliche Speicherdatei im Projektordner: `serienwertung_daten.json`.

## Versionsnummern

Ab v44 / v50 / v03 steht die Version sichtbar hinter der Überschrift und im
Fenstertitel (`<span class="versions-badge">`). Nummernkreise laufen je Datei
getrennt weiter.


## Rating-Stand (Serienwertung v09, Reiter Vereins-Rating)

"Rating-Stand exportieren" erzeugt `rating_stand_JJJJ-MM-TT.json` (Datum =
Stichtag). Die Turnierpläne lesen diese Datei ab Phase 4 für das Handicap.

```json
{
  "ratingStand": {
    "formatVersion": 1,
    "programm": "Billard Serienwertung v09",
    "erstelltAm": "2026-09-17T10:00:00.000Z",
    "stichtag": "2026-09-17",
    "zeitraumMonate": 12,
    "mindestRacks": 100,
    "rueckgriffMonate": 36,
    "gewichtStartwert": 30,
    "vereinsschnitt": 500,
    "disziplinen": ["8-Ball", "9-Ball", "10-Ball", "Multi-Ball"],
    "spieler": [
      {
        "name": "Olli",
        "schreibweisen": ["Olli", "Olly"],
        "disziplinen": {
          "8-Ball":     { "rating": 557, "racks": 87, "status": "vorlaeufig", "quelle": "vorläufig" },
          "9-Ball":     { "rating": 557, "racks": 0,  "status": "rueckgriff", "quelle": "aus 8-Ball" },
          "10-Ball":    { "rating": 557, "racks": 0,  "status": "rueckgriff", "quelle": "aus 8-Ball" },
          "Multi-Ball": { "rating": 557, "racks": 0,  "status": "rueckgriff", "quelle": "aus 8-Ball" }
        },
        "gesamt": { "rating": 557, "racks": 87, "status": "vorlaeufig", "quelle": "vorläufig" }
      }
    ]
  }
}
```

- `spieler`: alle Spieler mit Partien innerhalb des Rückgriffs sowie Spieler
  mit manuellem Startwert, ohne die ausgeblendeten Spieler. `name` ist die Schreibweise nach "Spieler
  zusammenführen", `schreibweisen` enthält alle bekannten Varianten für die
  Zuordnung im Turnierplan.
- `status`: `eigen` (Racks >= Mindest-Racks), `vorlaeufig` (weniger Racks),
  `rueckgriff` (keine Racks in der Disziplin, Wert aus den anderen
  Disziplinen), `manuell` (keine Racks, manueller Startwert), `schnitt`
  (keine Racks, Vereinsschnitt).
- `quelle`: Klartext wie in der Rating-Liste.

### Rechenverfahren

- Rack-Siegchance: `p = 1 / (1 + 2^((R_B - R_A) / 100))`.
- Maximum-Likelihood über alle gewerteten Racks im Fenster (MM-Verfahren für
  Bradley-Terry), Abbruch bei Änderung < 0,001 Punkte.
- Gezählt werden selbst gespielte Sätze: `satzA - vorgabeA`, `satzB - vorgabeB`.
- Nicht gezählt: Turniere und Partien mit `werten: false`, Partien, deren
  Spieler nach der Zusammenführung dieselbe Person sind.
- Startwert: je Spieler `gewichtStartwert` virtuelle Racks (je zur Hälfte
  gewonnen und verloren) gegen einen festen Gegner mit dem Startwert.
  Rangfolge: manueller Startwert, Rating aus allen anderen Disziplinen
  (ohne die gerechnete), Vereinsschnitt 500.
- Fenster: Partien ab Stichtag minus `zeitraumMonate`. Spieler mit weniger als
  `mindestRacks` bekommen ältere Partien dazu, höchstens bis Stichtag minus
  `rueckgriffMonate`. Eine Partie zählt, wenn sie im Fenster eines ihrer
  beiden Spieler liegt.

### Datenbestand der Serienwertung

- `DB_VERSION` 4: zusätzlich `ratingEinstellungen` (disziplin, zeitraum,
  mindestRacks, rueckgriff, gewicht), `ratingStartwerte`
  (`[{ name, wert }]`, Wert 100 bis 1000) und `ratingAusgeblendet`
  (`["Name", ...]`). Der Stichtag wird nicht gespeichert.
- Ältere Stände (v07: Version 1, v08: Version 2, v09: Version 3) werden beim
  Laden umgestellt; vorher wird
  `billard_serienwertung_vor_v10_<Datum>.json` heruntergeladen.

### Ausgeblendete Spieler (Serienwertung v10)

- `ratingAusgeblendet` enthält Spielernamen, die weder in der Rating-Liste noch
  im Rating-Stand erscheinen. Ihre Partien zählen weiter, die Ratings der
  Gegner bleiben also unverändert.
- Der Abgleich läuft über denselben Schlüssel wie die Zusammenführung, andere
  Schreibweisen desselben Spielers sind damit ebenfalls ausgeblendet.
- Ausblenden setzt einen ausgeschalteten Löschschutz voraus und ist über das
  Kreuz am Eintrag jederzeit umkehrbar.
- Sollen auch die Partien nicht mehr zählen, bleibt der Weg über das
  Turnier-Archiv: Turnier oder einzelne Partien abhaken.


## Handicap in den Turnierplänen (v53 / v56 / v66)

Die Turnierpläne lesen den Rating-Stand über "Rating-Stand laden" in der
Einstellungsleiste. Aus der Rating-Differenz entsteht je Paarung eine Vorgabe
in Sätzen, mit der der schwächere Spieler startet.

### Berechnung der Vorgabe

- Rack-Siegchance: `p = 1 / (1 + 2^(-d / 100))` mit
  `d = |Rating A - Rating B| * Stärke / 100`.
- Gewählt wird die Vorgabe `v` (0 bis `Race to - 1`, höchstens die
  eingestellte Obergrenze), bei der die Siegwahrscheinlichkeit des Stärkeren
  über die ganze Partie am nächsten bei 50 % liegt. Gerechnet wird exakt über
  den Race-Verlauf (negative Binomialverteilung), nicht nur je Rack.
- Die Vorgabe wird nicht gespeichert, sondern aus Ratings, Race to, Stärke und
  Obergrenze jederzeit neu gerechnet. Weil die Startnummern stabil sind, bleibt
  sie auch nach einem Neuaufbau des Spielplans (Spieler nachtragen) gleich.
- Je Phase gilt deren Race to: Gruppenphase, Platzierungsduelle, jede KO-Runde
  und Phase 3 werden getrennt gerechnet.

### Ratings im Turnierplan

- Zuordnung über `name` und `schreibweisen` des Rating-Stands, Groß- und
  Kleinschreibung sowie mehrfache Leerzeichen spielen keine Rolle.
- Nicht gefundene Spieler bekommen den Vereinsschnitt 500 und sind rot
  markiert. Jeder Wert lässt sich von Hand überschreiben.
- Mit der Spielart wechselt der Wert zur jeweiligen Disziplin.
- Beim ersten eingetragenen Ergebnis werden die Ratings eingefroren:
  Handicap-Einstellungen sind danach gesperrt, ebenso die Rating-Felder aller
  Spieler mit begonnenen Spielen. Nachzügler bekommen ihren Wert beim
  Nachtragen und werden sofort eingefroren.
- Maßgeblich ist, ob aktuell ein Ergebnis eingetragen ist, nicht der
  Turnierstart. Wird das letzte Ergebnis wieder zurückgenommen, sind
  Einstellungen und Rating-Felder erneut wählbar und die Ratings werden neu
  aus Stand und Namen bestimmt (`hcStandPruefen()` nach jeder Eingabe).
  Der Turnierstart für die Zeitprognose bleibt davon unberührt.

### Spielstand

- Die Satzfelder sind mit der Vorgabe vorbelegt (0:3 bei Vorgabe 3).
  Eingetragen wird der Endstand inklusive Vorgabe.
- Ein Spiel gilt erst als begonnen, wenn der Stand von der Vorgabe abweicht.
  Solche Spiele zählen nicht in der Live-Rangliste und lösen weder den
  Turnierstart noch die Sperre einer Runde aus.
- Punkte, Summe und Satz-Differenz rechnen mit dem Endstand inklusive Vorgabe.
  Zeitprognose und Vereins-Rating zählen nur selbst gespielte Sätze.
- Unter die Vorgabe lässt sich ein Feld nicht setzen.

### Turnierstand (Snapshot)

Neues Feld `handicap`, `STORAGE_VERSION` bleibt unverändert. Ältere Stände ohne
das Feld laufen ohne Handicap weiter.

```json
"handicap": {
  "aktiv": true,
  "staerke": 75,
  "obergrenze": 0,
  "versteckt": false,
  "stand": { "stichtag": "2026-09-17", "datei": "rating_stand_2026-09-17.json",
             "spieler": [ { "name": "Olli", "schreibweisen": ["Olli", "Olly"],
                            "werte": { "8-Ball": { "rating": 557, "status": "eigen", "quelle": "eigene Daten" } } } ] },
  "manuell": { "3": 600 },
  "fest": { "0": { "rating": 642, "status": "eigen", "quelle": "eigene Daten" } }
}
```

- `manuell`: von Hand gesetzte Ratings je Startnummer.
- `fest`: beim Turnierstart eingefrorene Werte je Startnummer.
- `staerke` in Prozent, `obergrenze` 0 = keine.
- `versteckt`: Schaltfläche "Rating ausblenden" in der Einstellungsleiste
  (v57 / v61 / v71). Blendet Ratings in Spielerliste, Spielplan, Rangliste,
  Bericht und Mail aus; die Vorgabe wird unabhängig davon weiter berechnet.
  Jederzeit umschaltbar, auch während das Turnier läuft. Ältere Stände ohne
  das Feld gelten als `false`.


## Spieler-Detail in der Rating-Liste (Serienwertung v11)

Ein Lupe-Symbol am Ende jeder Zeile öffnet je Spieler eine Detailzeile mit
zwei Kacheln. Es ist jeweils nur ein Spieler gleichzeitig aufgeklappt.

- **Partienverlauf:** alle Partien des Spielers, die im aktuell berechneten
  Fenster liegen (inklusive einer möglichen Erweiterung), neueste zuerst.
  Datum, Gegner, Phase und Stand aus Sicht des Spielers, mit der Vorgabe als
  Zusatz, falls vorhanden. Diese Zusatzfelder (Turniername, Phase, Original-
  stand) hängen an den Partien aus `ratingPartien()`, verändern aber nicht die
  Rechnung selbst.
- **Startwert-Einfluss:** Quelle und Höhe des Startwerts sowie ein Balken, der
  zeigt, zu wie viel Prozent der Startwert noch mitzählt
  (`Gewicht / (Gewicht + Racks)`, bezogen auf die eigenen Racks in der
  aktuell gerechneten Ansicht).
- Ausgeblendete Spieler haben kein Lupe-Symbol. Sie können aber in fremden
  Partienverläufen als Gegner erscheinen, weil ihre Partien weiterhin zählen.
