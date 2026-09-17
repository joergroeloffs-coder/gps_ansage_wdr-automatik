# GPS-Ansageautomatik

Web-App für automatische Bordansagen auf Basis der GPS-Position (Einlaufen/Ablegen im Hafen).

## Nutzung

1. `index.html` im Browser öffnen (z.B. per lokalem Webserver, da `stations.json` per `fetch` geladen wird — direktes Öffnen als `file://` funktioniert in den meisten Browsern nicht wegen CORS). Über GitHub Pages entfällt das (siehe unten).
   - Schnelltest lokal: im Projektordner z.B. `python3 -m http.server 8000` ausführen und `http://<handy-ip>:8000` öffnen.
2. Standortzugriff erlauben.
3. "GPS-Tracking starten" drücken.

**Wichtig:** Die Seite muss während der Fahrt im Vordergrund geöffnet bleiben (nicht in den Hintergrund schicken, Handy nicht sperren) – Browser pausieren GPS und Sprachausgabe sonst. Die App aktiviert beim Start automatisch einen Wake Lock, der das Display wach hält, solange getrackt wird (unterstützt von Chrome/Edge auf Android; Safari/iOS unterstützt das aktuell nicht – dort Display-Sperre manuell deaktivieren).

## Konfiguration (`stations.json`)

- `fleet`: die vier Schiffe der Flotte, je mit `key` (interner Schlüssel, muss zu den Schiffskürzeln im Fahrplan passen: `NORDFRIESLAND`, `NORDERAUE`, `SCHLESWIG - HOLSTEIN`, `UTHLANDE`), `audioKey` (Dateinamens-Bestandteil für die Ablege-Audiodateien) und `name` (gesprochener Anzeigename, ersetzt `{schiff}`). Im UI oben wählt die Crew das aktuelle Schiff aus (persistiert im Browser) – das bestimmt sowohl den gesprochenen Schiffsnamen als auch, welcher Fahrplan-Eintrag für die Automatik (siehe unten) herangezogen wird.
- `defaultShipKey`: Schiff, das beim ersten Öffnen (ohne gespeicherte Auswahl) vorausgewählt ist.
- `textTemplates.arrival`: Objekt mit einer Vorlage je Anleger-Typ – `ohneSeitenausstieg` und `mitSeitenausstieg`. Platzhalter `{hafen}` wird automatisch durch den Stationsnamen ersetzt. Fehlt eine Vorlage (z.B. `mitSeitenausstieg: null`), wird für Stationen ohne eigenen Text keine Ansage abgespielt (siehe Log-Hinweis).
- `textTemplates.departure`: Fallback-Ablege-Vorlage für Stationen ohne eigene `departureRoutes` (siehe unten).
- `stations`: Liste der Häfen/Stationen mit `lat`/`lon` und Einlaufen-Radius. `arrival.texts.<berthType>` überschreibt pro Station die globale Vorlage für einen bestimmten Anleger-Typ (z.B. weil der Hafen eine eigene Formulierung braucht).
- `arrival.radiusMeters`: Ab dieser Entfernung zum Hafen wird die Einlaufen-Ansage ausgelöst.
- **Anleger-Typ-Auswahl (mit/ohne Seitenausstieg)**: GPS kann die beiden Anleger je Hafen nicht unterscheiden, da sie zu nah beieinander liegen. Im UI wählt die Crew daher pro Station manuell per Dropdown "Mit Seitenausstieg" (Standard) / "Ohne Seitenausstieg", bevor der Hafen angelaufen wird. Die Automatik löst weiterhin per GPS aus, spielt aber den zur Auswahl passenden Text.
- **`departureRoutes`** (pro Station): Liste möglicher Fahrtziele ab diesem Hafen, je mit `id`, `label` (Anzeige im Dropdown) und `text` (Ansagetext, Muster "Wir begrüßen Sie an Bord der {schiff}. Wir legen jetzt ab zur Überfahrt nach/über … {ziel}."). Da mehrere Routen möglich sind (z.B. ab Dagebüll nach Wyk, nach Wittdün, oder über Wyk nach Wittdün), wählt die Crew im UI vor dem Ablegen per Dropdown "Fahrtziel für die nächste Abfahrt" die passende Route – die Ablege-Ansage (Text und Audiodatei) richtet sich danach.
- `hysteresisFactor`: Verhindert Mehrfachauslösung durch GPS-Schwankungen am Radius-Rand (gilt für Einlaufen).
- `departureDetection`: globale Ablege-Erkennung (nicht pro Station, siehe unten):
  - `stableRadiusMeters` (Default 35): Umkreis, in dem das Schiff als "still liegend" gilt. Bewusst großzügig, da GPS auf einem Stahlschiff deutlich ungenauer ist als an Land (oft 20–50m statt 5–10m Fehler).
  - `stableDurationMinutes` (Default 7): So lange muss das Schiff innerhalb von `stableRadiusMeters` bleiben, damit die Position als Ablege-Anker gilt ("angelegt").
  - `driftToleranceSeconds` (Default 60): Kurze GPS-Ausreißer über `stableRadiusMeters` hinaus werden bis zu dieser Dauer ignoriert (Messung läuft weiter), statt die 7-Minuten-Messung sofort neu zu starten. Erst wenn die Position durchgehend länger als diese Zeit außerhalb bleibt, gilt das als echte Bewegung.
  - `departureRadiusMeters` (Default 50): Entfernung vom Anker, ab der ein Ablegen erkannt wird.
  - `departureWindowMinutes` (Default 2): Diese Entfernung muss innerhalb dieses Zeitfensters erreicht werden – sonst gilt es als langsames Wegdriften statt echtem Ablegen, und es wird nichts ausgelöst.
  - `stationMatchRadiusMeters` (Default 300): Umkreis, in dem der erkannte Ablege-Anker einer konfigurierten Station zugeordnet wird (für den passenden Ansagetext).
  - `genericText`: Fallback-Ansagetext, falls der Anker keiner Station zugeordnet werden kann.
- `secondaryAnnouncements`: manuell auslösbare Zusatzansagen, unabhängig von GPS (z.B. `autodeckFreigabe` – Crew drückt im UI einen eigenen Button, sobald das Autodeck zum Verlassen freigegeben werden soll).
- `fahrplanScheduleUrl`: URL zur `fahrplan_schedule.json` im `dienstplan`-Repo (siehe nächster Abschnitt).
- `autoTracking.leadMinutes` / `autoTracking.lagMinutes`: wie viele Minuten vor der ersten bzw. nach der letzten heutigen Abfahrt des gewählten Schiffs das automatische GPS-Tracking bereits läuft bzw. noch weiterläuft.

**Wichtig:** Die Koordinaten in `stations.json` sind Platzhalter und müssen durch die echten Hafenpositionen der Wikingerdampfschiffsreederei ersetzt werden. Ebenso sind die Ansagetexte nur Beispiele.

## Fahrplan-Automatik (Schiffsauswahl + automatisches GPS-Tracking)

Im UI oben wählt die Crew das aktuelle Schiff (eines der vier: Nordfriesland, Norderaue, Schleswig-Holstein, Uthlande). Das bestimmt:

1. Den gesprochenen Schiffsnamen in den Ablege-Ansagen (Platzhalter `{schiff}`).
2. Welche vorproduzierte Ablege-Audiodatei verwendet wird (`departure_<hafen>_<route>_<audioKey>.mp3`). **Aktuell liegen nur Audiodateien für "Schleswig-Holstein" vor** – für die anderen drei Schiffe greift automatisch der Live-TTS-Fallback (mit korrektem Schiffsnamen, aber Gerätestimme), bis eigene Audiodateien erzeugt werden.
3. Welcher Fahrplan-Eintrag für die Automatik herangezogen wird.

**Automatikmodus** (Checkbox im Abschnitt "Fahrplan-Automatik"): Wenn aktiviert, lädt die App periodisch `fahrplan_schedule.json` aus dem `dienstplan`-Repo (Cross-Origin-Fetch von `raw.githubusercontent.com` – dort wird sie stündlich per GitHub Action aus dem echten Fahrplan-PDF von faehre2.de neu erzeugt, Skript `fahrplan_export_gps.py` im `dienstplan`-Repo). Für das gewählte Schiff werden die heutigen Abfahrten ermittelt; das "Einsatzfenster" reicht von `leadMinutes` vor der ersten bis `lagMinutes` nach der letzten Abfahrt. Innerhalb dieses Fensters wird GPS-Tracking automatisch gestartet, außerhalb automatisch gestoppt (spart Akku, wenn das Schiff nicht fährt). Der Status wird live angezeigt ("heute im Einsatz von X bis Y").

**Automatische Fahrtziel-Erkennung:** Sobald "Anlegen" erkannt wird (siehe Ablegen-Logik unten), schaut die App im Fahrplan nach, welche Abfahrt für das gewählte Schiff als nächstes ab diesem Hafen ansteht, und wählt die passende Route im Dropdown "Fahrtziel für die nächste Abfahrt" automatisch aus – manuelles Nachjustieren bleibt weiterhin möglich. Dazu haben Stationen ein `fahrplanHafen`-Feld (Kurzname wie im Fahrplan-PDF: "Dagebüll", "Wyk", "Wittdün") und Routen ein `fahrplanMatch.direkt` (true/false): eine als "direkt" markierte Abfahrt ab Dagebüll oder Wittdün bedeutet, das Schiff fährt ohne Zwischenstopp durch bis zum jeweils anderen Endhafen – das wird auf die entsprechende "über Wyk..."-Route gemappt, eine nicht-direkte Abfahrt auf die einfache Route bis Wyk. Ohne passenden Fahrplan-Eintrag (z.B. Fahrplan nicht geladen, oder keine Abfahrt mehr gefunden) bleibt die zuletzt gewählte bzw. die Standard-Route stehen.

**Ablege-Ansage per Fahrplan-Zeit (unabhängig von GPS):** Solange die App nicht durchgehend im Hintergrund laufen kann, reicht die GPS-Bewegungserkennung (7 Minuten Stillstand + Wegfahren) oft nicht aus, um zuverlässig auszulösen. Als Ergänzung prüft die App jede Minute alle heutigen Fahrplan-Abfahrten des gewählten Schiffs: eine Minute vor der geplanten Abfahrtzeit (Systemzeit des Geräts) wird die Ablege-Ansage direkt anhand des Fahrplans ausgelöst, inklusive automatischer Routen-Erkennung (siehe oben) – ganz ohne dass GPS die Bewegung erkannt haben muss. Ein Dedupe-Mechanismus verhindert, dass GPS-Erkennung und Fahrplan-Zeit-Trigger dieselbe Abfahrt doppelt ansagen (20 Minuten Sperre je Hafen nach einer Ansage). Voraussetzung ist weiterhin, dass die Seite zum fraglichen Zeitpunkt geöffnet ist – die Systemzeit-Prüfung braucht dafür aber keine ununterbrochene GPS-Historie wie die Bewegungserkennung.

**Wichtig:** Die Automatik ersetzt nicht die manuellen Start/Stopp-Buttons – bei deaktiviertem Automatikmodus verhält sich die App wie zuvor. Der Fahrplan-Export läuft nur auf dem `main`-Branch von `dienstplan` (GitHub-Actions-Zeitpläne feuern nicht auf Feature-Branches) – solange die entsprechende Änderung dort nicht gemerged ist, bleibt `fahrplan_schedule.json` leer/fehlt, und die Automatik zeigt "Fahrplan konnte nicht geladen werden" bzw. bleibt inaktiv.

## Referenzpunkte per Karte setzen (`karte.html`)

Statt Koordinaten manuell zu suchen: `karte.html` im Browser öffnen (funktioniert auch direkt als Datei, keine `fetch`-Abhängigkeit). Zeigt eine OpenStreetMap-Karte, zentriert auf das nordfriesische Wattenmeer.

1. Auf die Karte klicken → setzt einen Referenzpunkt (Marker ist verschiebbar).
2. In der Tabelle darunter Name sowie Annäherungs-/Ablege-Radius je Punkt eintragen.
3. "JSON exportieren" klicken → erzeugt ein fertiges `stations.json`-Gerüst (Texte sind Platzhalter nach Schema "In Kürze legen wir in … an." bzw. "Wir legen jetzt in … ab.").
4. Per "In Zwischenablage kopieren" übernehmen und in `stations.json` einfügen, Texte final anpassen.

Benötigt Internetzugang zum Laden der Kartenkacheln (OpenStreetMap) und der Leaflet-Bibliothek (CDN).

## Ansagetexte aus Bausteinen zusammensetzen (`baukasten.html`)

Statt jedes Mal einen kompletten Ansagetext zu diktieren: `baukasten.html` im Browser öffnen (funktioniert direkt als Datei, kein Server nötig). Enthält wiederverwendbare Textbausteine (Begrüßung, PKW-Hinweis, Fußgänger-Varianten, Schlusssatz, Ablegen, Sonstiges).

1. Hafenname wählen (ersetzt `{hafen}` in den Bausteinen).
2. Passende Bausteine per "+ Hinzufügen" in die gewünschte Reihenfolge bringen (Pfeile zum Verschieben, ✕ zum Entfernen).
3. Ergebnis unten lesen, per "Vorhören" mit der Gerätestimme testen.
4. Eigene neue Bausteine (z.B. weitere Eventualitäten) über das Formular unten hinzufügen – bleiben im Browser gespeichert (localStorage), auch nach einem Neuladen.
5. Fertigen Text per "Text kopieren" übernehmen und zur Aufnahme/Integration weitergeben (z.B. hier im Chat einfügen) – die eigentliche MP3-Erzeugung (Piper) und Einbindung in `stations.json` erfolgt weiterhin über den Chat, da Piper nicht im Browser läuft.

Die Bausteine sind nur Startvorschläge und können beliebig ergänzt oder gelöscht werden – die Vorgaben aus den bisherigen Ansagetexten sind als Standard-Bausteine bereits enthalten.

## Funktionsweise

- **Einlaufen**: klassische Geofence-Annäherung — sobald die Distanz zum Hafen den Annäherungsradius unterschreitet, wird die Ansage einmalig abgespielt. Erst wenn das Schiff die Zone wieder deutlich verlässt, wird der Trigger erneut "scharf geschaltet" (Hysterese).
- **Ablegen**: dynamische Anker-Erkennung, unabhängig von vorgegebenen Koordinaten. Bleibt das Schiff länger als `stableDurationMinutes` innerhalb von `stableRadiusMeters` an einer Position, gilt diese Position als Ablege-Anker ("angelegt"). Entfernt sich das Schiff danach innerhalb von `departureWindowMinutes` um mehr als `departureRadiusMeters` von diesem Anker, wird die Ablege-Ansage ausgelöst. Entfernt es sich stattdessen langsam über einen längeren Zeitraum (z.B. Drift durch Tide/Wind), wird nichts ausgelöst und die Erkennung setzt sich zurück.
- **Sprachausgabe**: Es wird immer zuerst versucht, eine vorproduzierte Audiodatei aus dem Ordner `audio/` abzuspielen (siehe unten). Existiert die Datei nicht, fällt die App automatisch auf die Web Speech API (geräteeigene TTS-Engine) zurück – dabei kann oben im UI unter "Stimme für Ansagen" zwischen den auf dem Gerät installierten deutschen Stimmen gewählt werden. Das ist nur ein Notbehelf: Qualität und Akzent hängen stark vom Gerät ab.

## Vorproduzierte Audiodateien (`audio/`)

Alle Dateien liegen bereits im Ordner `audio/`, erzeugt mit [Piper](https://github.com/rhasspy/piper) (kostenlose, offline laufende neuronale TTS, Stimme "Thorsten", CC0-Lizenz) – keine Kosten, kein Internet zur Laufzeit nötig. Wer eine andere/bessere Stimme möchte, kann die Dateien jederzeit ersetzen (z.B. über Azure Speech Studio, ElevenLabs oder Google Cloud TTS – alle mit kostenlosem Testkontingent – oder als echte Sprachaufnahme). Namenskonvention:

- Einlaufen: `arrival_<stationId>_<berthType>.mp3` (`berthType` = `ohneSeitenausstieg` oder `mitSeitenausstieg`)
- Ablegen (Stationen mit `departureRoutes`): `departure_<stationId>_<routeId>.mp3` – ein `routeId` pro Fahrtziel/Route, siehe `stations.json`
- Ablegen (Stationen ohne `departureRoutes`, Fallback): `departure_<stationId>.mp3`, sonst `departure_generic.mp3`
- Zusatzansagen: `secondary_<name>.mp3` (aktuell `secondary_autodeck_freigabe.mp3`)

Aktuell vorhanden:

| Datei | Inhalt |
|---|---|
| `arrival_dagebuell_ohneSeitenausstieg.mp3` | Einlaufen Dagebüll, ohne Seitenausstieg |
| `arrival_dagebuell_mitSeitenausstieg.mp3` | Einlaufen Dagebüll, mit Seitenausstieg |
| `arrival_wyk-auf-foehr_ohneSeitenausstieg.mp3` | Einlaufen Wyk auf Föhr, ohne Seitenausstieg |
| `arrival_wyk-auf-foehr_mitSeitenausstieg.mp3` | Einlaufen Wyk auf Föhr, mit Seitenausstieg |
| `arrival_wittduen-auf-amrum_ohneSeitenausstieg.mp3` | Einlaufen Wittdün auf Amrum, ohne Seitenausstieg |
| `arrival_wittduen-auf-amrum_mitSeitenausstieg.mp3` | Einlaufen Wittdün auf Amrum, mit Seitenausstieg |
| `departure_dagebuell_dagebuell-wyk.mp3` | Ablegen Dagebüll → Wyk auf Föhr |
| `departure_dagebuell_dagebuell-wittduen.mp3` | Ablegen Dagebüll → Wittdün auf Amrum |
| `departure_dagebuell_dagebuell-via-wyk-wittduen.mp3` | Ablegen Dagebüll → über Wyk auf Föhr nach Wittdün auf Amrum |
| `departure_wyk-auf-foehr_wyk-dagebuell.mp3` | Ablegen Wyk auf Föhr → Dagebüll |
| `departure_wyk-auf-foehr_wyk-wittduen.mp3` | Ablegen Wyk auf Föhr → Wittdün auf Amrum |
| `departure_wittduen-auf-amrum_wittduen-dagebuell.mp3` | Ablegen Wittdün auf Amrum → Dagebüll |
| `departure_wittduen-auf-amrum_wittduen-wyk.mp3` | Ablegen Wittdün auf Amrum → Wyk auf Föhr |
| `departure_wittduen-auf-amrum_wittduen-via-wyk-dagebuell.mp3` | Ablegen Wittdün auf Amrum → über Wyk auf Föhr nach Dagebüll |
| `departure_generic.mp3` | Ablegen, falls kein Hafen zugeordnet werden konnte |
| `secondary_autodeck_freigabe.mp3` | Manuelle Zusatzansage "Autodeck-Freigabe" |

**Cache-Busting:** Die App hängt an jede Audio-URL `?v=<audioVersion>` an (Wert aus `stations.json`). Nach dem Ersetzen/Aktualisieren von MP3-Dateien `audioVersion` in `stations.json` um 1 erhöhen – sonst kann es sein, dass Browser oder GitHub-Pages-CDN noch die alte, zwischengespeicherte Version abspielen, obwohl die Datei im Repo schon aktuell ist.

Die exakten Texte für jede Datei stehen in `stations.json` (`arrival.texts`, `departureRoutes`, `textTemplates`, `secondaryAnnouncements` – Platzhalter `{hafen}` durch den jeweiligen Hafennamen ersetzen). Hochladen entweder per `git`, oder direkt über die GitHub-Weboberfläche: Ordner `audio/` öffnen → "Add file" → "Upload files" → Dateien reinziehen → Commit.

Fehlt eine Datei (z.B. bei einem neu hinzugefügten Hafen), spielt die App automatisch die Gerätestimme mit dem Text ab und vermerkt das im Log – nichts bricht dadurch ab.

## Testmodus

Im UI gibt es unter "Erweitert: Testmodus & Diagnose" (einklappbar, unten auf der Seite) einen Testmodus mit manueller Eingabe von Position und Geschwindigkeit, um die Auslöselogik ohne echte Fahrt zu testen, sowie einen TTS-Testknopf.

## Oberfläche / Design

Helles Design (weißer Hintergrund, blauer Akzent), große Buttons für die Bedienung mit Handschuhen/bei Sonnenlicht. Protokoll-/Log-Ausgaben (pro Hafen, Ablegen, Fahrplan) sind standardmäßig ausgeblendet – Checkbox "Protokoll/Diagnose anzeigen" oben schaltet sie sichtbar. Der Testmodus liegt in einem einklappbaren "Erweitert"-Bereich ganz unten, damit die normale Bedienung (Schiff, Anleger, Fahrtziel, Start/Stopp, Status) nicht durch Debug-Informationen überladen wird.

## Offene Punkte / nächste Schritte

- Ansagetexte für Ablegen (Vorlage gilt bisher pauschal, keine Anleger-Typ-Unterscheidung) ggf. noch anpassen.
- Weitere "Begebenheiten" (über Einlaufen/Ablegen hinaus) als zusätzliche Einträge in `stations.json` bzw. als eigener Ereignistyp ergänzen, sobald definiert.
- Test auf echtem Android-Gerät/Schiff zur Kalibrierung von Radien und Zeitfenstern.
