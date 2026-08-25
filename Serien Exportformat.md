\# Serien-Exportformat (Stand v44 / v50 / Serienwertung v03)



Alle drei Turnierpläne schreiben ihrem JSON-Export seit v43 bzw. v49 einen

zusätzlichen Block `serie` bei. Nur dieser Block wird von der Serienwertung

gelesen; die Auswertungsseite rechnet keine Ranglisten nach.



```json

"serie": {

&#x20; "formatVersion": 1,

&#x20; "modus": "einzelgruppe" | "zwei-gruppen" | "gruppen-ko",

&#x20; "disziplin": "9-Ball",

&#x20; "datum": "2026-06-07",

&#x20; "teilnehmer": 14,

&#x20; "vollstaendig": true,

&#x20; "ranking": \[ { "platz": 1, "name": "Volker" }, ... ]

}

```



\- `datum`: Zeitpunkt des ersten eingetragenen Ergebnisses (`tournamentStart`),

&#x20; ersatzweise das aktuelle Datum. In der Auswertungsseite je Turnier änderbar.

\- `vollstaendig`: false, solange Spiele offen sind bzw. Phase 2 / KO-Runde nicht

&#x20; gestartet oder nicht entschieden ist. Solche Turniere werden nur mit

&#x20; ausdrücklichem Häkchen eingelesen.

\- `ranking`: fertige Endplatzierung. Im KO-Modus können sich mehrere Spieler

&#x20; einen Platz teilen (dichte Platzvergabe), z.B. 1,2,3,4,5,5,5,5,9,9,9,9,...

&#x20; Geteilte Plätze ergeben in der Serienwertung gleiche Punkte.



Erzeugt wird der Block in jeder Datei durch `buildSerienExport()`, eingebunden

in `getCurrentSnapshot()` innerhalb eines try/catch, damit ein Fehler dort das

Speichern des Turnierstandes nicht verhindert.



\## Punkteformel der Serienwertung



`punkte = teilnehmer + 1 - platz`, der Sieger erhält einen Zusatzpunkt.

Entspricht der bisherigen Excel-Formel

`=WENN(V10="";"";V$39+1-V10+WENN(V10=1;1;0))`.



Reihenfolge bei Punktgleichheit: mehr erste Plätze, dann mehr zweite Plätze

usw., zuletzt der Name. Punktgleiche Spieler teilen sich einen Platz.



Streichergebnisse sind je Serie einstellbar ("beste X von N"); gestrichen

werden die punktschwächsten Turniere eines Spielers.



\## Browser: Chrome oder Edge, nicht Firefox



Am 25.08.2026 auf dem Notebook geprüft: Firefox behält den localStorage einer

per `file://` geöffneten Datei nicht über das Schließen hinaus, Edge tut es.

Damit sind in Firefox die automatische Sicherung und die Restore-Punkte aller

drei Turnierpläne wirkungslos. Ab v44 / v50 zeigen die Turnierpläne deshalb

einen roten Warnkasten, sobald sie in Firefox laufen (Erkennung über

`navigator.userAgent`).



\## Speicherdatei der Serienwertung



Ab v02 kann die Serienwertung ihren Datenbestand in eine echte Datei schreiben

(File System Access API, `showSaveFilePicker` / `showOpenFilePicker`, Dateiverweis

in IndexedDB unter `billard-serienwertung/handles/speicherdatei`). Einmal

verknüpfen, danach schreibt die Seite jede Änderung selbst hinein; beim nächsten

Öffnen ist eine Nutzeraktion nötig (Banner mit Schaltfläche "Zugriff erlauben und

laden"), das verlangt der Browser.



Firefox kennt diese Schnittstelle nicht. Dort erscheint stattdessen ein Hinweis,

und es bleibt bei Export und Import von Hand.



Übliche Speicherdatei im Projektordner: `serienwertung\_daten.json`.



\## Versionsnummern



Ab v44 / v50 / v03 steht die Version sichtbar hinter der Überschrift und im

Fenstertitel (`<span class="versions-badge">`). Nummernkreise laufen je Datei

getrennt weiter.

