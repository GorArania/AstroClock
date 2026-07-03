# AstroClock — Projektkontext für Claude Code

Dieses Dokument fasst den aktuellen Stand des Projekts zusammen, damit direkt
weitergearbeitet werden kann, ohne frühere Design-Entscheidungen versehentlich
rückgängig zu machen. Viele Details hier wurden über mehrere Iterationen mit
Testen und Nachrechnen erarbeitet — insbesondere die als "NICHT ÄNDERN OHNE
RÜCKFRAGE" markierten Abschnitte sind bewusste Entscheidungen, keine Zufälle
oder unfertigen Zwischenstände.

## Projektüberblick

Eine astronomisch korrekte 24-Stunden-Uhr als Web-App (Vorstufe zu einer
Android-App in einer separaten, späteren Phase — siehe "Scope" unten). Die
Sonne fungiert als Stundenzeiger, während sich der komplette Sternenhimmel
passend dreht. Kein App-Store-Launch als Ziel — eine funktionierende, schön
aussehende Anzeige mit kaum Benutzerinteraktion.

**Live-Deployment:** `goraran.de/astroclock` (dieser Raspberry Pi)
**Datei:** `/var/www/html/astroclock/index.html`
**Webserver:** Apache2 (vhost-Configs: `/etc/apache2/sites-enabled/goraran.de.conf`
und `goraran.de-ssl.conf`)

---

## Kernkonzept: Anzeigemodell "Modell A+B"

Das ist das Herzstück des Projekts. Zwei Komponenten, die NICHT unabhängig
voneinander geändert werden dürfen, ohne die Konsequenzen zu verstehen:

**Festes Grundgerüst (ändert sich nie):**
24h-Zifferblatt, 0/24 oben, 6 rechts, 12 unten, 18 links. Reine bürgerliche
Uhrzeit (Zeitzone + Sommerzeit aus dem System).

**Sonne als Stundenzeiger:**
- Winkel = exakte aktuelle Uhrzeit (NICHT der echte Azimut!) — sitzt immer
  exakt auf ihrer Stundenmarke
- Radius = `(90° − |Höhe|) / 90° × R` → Zenit/Nadir = Mitte, Horizont = Rand
- Tags hell, nachts verschattet (gleiche Position, andere Farbe)

**Sternhintergrund (inkl. Mond, Planeten, Sternbilder) dreht sich passend dazu:**
```
Rotation = Uhrzeit-Winkel(Sonne) − echter Azimut(Sonne)
Position_auf_Scheibe(Objekt) = Azimut(Objekt) + Rotation
```

**Bewusst akzeptierte Konsequenz: Norden pendelt im Tagesverlauf** (am
Standort Zwenkau, 51.22°N, zwischen ca. −1° und +40°). Das liegt daran, dass
der Sonnen-Azimut nicht gleichförmig über den Tag läuft (Projektion der
gleichförmigen Stundenwinkel-Bewegung auf den Horizont ist nicht linear —
nahe Horizont morgens/abends langsam, nahe Süden/Mittag schnell). Gleiches
Prinzip wie bei den krummen Stundenlinien klassischer Sonnenuhren. Das ist
KEIN Bug, sondern eine gewollte und nachgerechnete Eigenschaft.

**Verifizierte Eigenschaft:** Legt man die Uhr flach hin und richtet den
Sonnenzeiger exakt auf die echte Sonne aus, zeigt der N-Marker exakt nach
echtem geografischem Norden — eine direkte mathematische Konsequenz der
Rotationsformel (kein Zufall, wurde durchgerechnet und bestätigt).

### Wichtige Formeln (Kern-Pipeline, wird für Sterne/Mond/Planeten geteilt)

```js
// Stunde (0..24) → Bildschirm-Koordinate
function hourToXY(h, radius, cx, cy) {
  const phi = (h / 24) * 2 * Math.PI;   // Uhrzeigersinn, ab "oben"
  return [cx + radius * Math.sin(phi), cy - radius * Math.cos(phi)];
}

// Äquatoriale Koordinaten (RA/Dec) → Horizontal (Alt/Az), braucht Lokale Sternzeit (LST)
function equatToHoriz(ra_h, dec_deg, lstDeg, lat_deg) { ... }

// Himmelsposition (Az/Alt) + Rotation → Bildschirm-Koordinate
function skyToDialXY(azDeg, altDeg, rotation, cx, cy, R) {
  const clockAngle = normalizeAngle(azDeg * Math.PI / 180 + rotation);
  const h = (clockAngle / (2 * Math.PI)) * 24;
  const r = (90 - Math.abs(altDeg)) / 90 * R;
  return hourToXY(h, r, cx, cy);
}
```

---

## NICHT ÄNDERN OHNE RÜCKFRAGE: Keine Bahnlinien mehr — pendelnde Auf-/Untergangs-Labels

**Frühere Design-Entscheidung (verworfen):** Es gab einmal zwei Bahn-Varianten
("B" uhrzeit-treu für die Sonne, "A streng" richtungs-treu für den Mond), die
als gezeichnete Verlaufslinie über die Scheibe liefen. Diese Bahnlinien wurden
**ersatzlos entfernt**: Wegen des im Tagesverlauf pendelnden Nordens (siehe
oben) wirkten die Linien irreführend/missverständlich für den Betrachter.

**Aktuelle Lösung:** Für Sonne UND Mond wird nur noch ein Zeit-Label am Rand
angezeigt (`↑ HH:MM` / `↓ HH:MM`), analog zum früheren Mond-Label. Die
Position dieses Labels:
- nutzt den **echten, festen Azimut** des Auf-/Untergangs-Ereignisses
  (berechnet einmalig aus SunCalc-Position zum exakten Ereigniszeitpunkt),
- wird aber jeden Frame mit der **aktuellen** Rotation auf die Scheibe
  projiziert (`skyToDialXY(azDeg, 0, rotation, ...)`) — genau die gleiche
  Rotation, die auch der Kompass benutzt.

**Ergebnis:** Das Label "pendelt" im Tagesverlauf exakt synchron mit den
Kompass-Himmelsrichtungen mit (weil beide dieselbe Rotation zur Positions-
berechnung verwenden). Der Nutzer kann so Ort (Himmelsrichtung) UND Zeit des
Auf-/Untergangs direkt ablesen, ohne dass eine (missverständliche) Linie über
die Scheibe gezogen wird.

Betroffene Funktionen: `computeSunRiseSet`, `computeMoonRiseSet`,
`computeRiseSetLabelPos`, `drawRiseSetLabel` (ersetzen die früheren
`computeSunPath`, `computeMoonPath`, `drawPath`, `drawPathEndpointLabel`).
Bitte hier nicht wieder eine Bahnlinie einführen, ohne das explizit
abzusprechen — das war ein bewusster Rückbau auf Nutzerwunsch, kein
Zwischenstand.

**Kollisionsfall Voll-/Neumond:** Nahe Neumond gehen Sonne und Mond am
(fast) selben Azimut auf/unter, nahe Vollmond kann es je nach Jahreszeit
ähnlich eng werden — dann würden sich die beiden Text-Labels decken.
Lösung in `resolveLabelCollisions`: Der Markierungspunkt bleibt IMMER exakt
an der echten Position (das ist gewollt und korrekt, auch wenn beide Punkte
dadurch übereinanderliegen) — nur die Text-Beschriftung wird bei
Unterschreiten des Mindestabstands (gemessene Textbreite + Puffer) entlang
der Tangente des Horizontrings auseinandergeschoben, nicht radial und nicht
zufällig. Bitte diesen Mechanismus beibehalten, falls weitere Labels
(z. B. Planeten-Auf-/Untergänge) hinzukommen sollten.

**Wichtig — Ausweich-Richtung anhand der Punkte, nicht der Array-Reihenfolge:**
Eine erste Version entschied "Label A weicht −Tangente, Label B weicht
+Tangente aus" rein nach der Reihenfolge im Array. Das konnte dazu führen,
dass z. B. der Mond-Text UNTER dem Sonnen-Text landete, obwohl der
Mond-PUNKT tatsächlich weiter oben lag als der Sonnen-Punkt — verwirrend,
weil Text-Reihenfolge und Punkt-Reihenfolge dann nicht mehr zusammenpassten.
`resolveLabelCollisions` bestimmt die Ausweich-Richtung jetzt anhand der
Projektion der (unveränderlichen) Punkt-Positionen auf die Tangente — das
Label, dessen Punkt weiter in eine Richtung liegt, weicht auch mit seinem
Text in genau diese Richtung aus. Bitte bei Änderungen an dieser Funktion
sicherstellen, dass diese Punkt-Text-Konsistenz erhalten bleibt.

**Kollisionsfall Kompass-Beschriftung:** Die Kompass-Buchstaben (N/NO/O/…)
liegen auf `R * 0.91`, die Auf-/Untergangs-Labels auf fast demselben Radius
(`R` minus Einwärts-Versatz) — bei Ereignissen nahe einer Himmelsrichtung
schrieb der Text bisher die Kompass-Buchstaben zu. Lösung in
`resolveCompassCollisions`: Die Kompass-Position ist das feste Referenzraster
und bleibt unverändert; das kollidierende Auf-/Untergangs-Label wird
stattdessen weiter nach innen (Richtung Zentrum) verschoben, bis es die
Bounding-Box des Kompass-Buchstabens nicht mehr überlappt. Der
Markierungspunkt bleibt wie immer exakt an der echten Position — nur der
Text weicht aus. `drawCompass` bekommt die Positionen aus
`computeCompassLabelPositions` übergeben (einmal pro Frame berechnet), damit
Zeichnen und Kollisionsprüfung dieselbe Geometrie verwenden.

**Wichtig — analytisch, nicht iterativ mit festem Schritt/Budget:** Eine
erste Version schob in kleinen festen Schritten (z. B. 8× `size*0.01`) nach
innen, bis keine Kollision mehr erkannt wurde. Das reichte je nach Tagesdatum
manchmal nicht aus (Label blieb teilweise unter dem Kompass-Buchstaben
verdeckt, ohne dass ein Fehler sichtbar war — wirkte wie "Zeit fehlt
komplett"). `resolveCompassCollisions` berechnet den nötigen Schub jetzt
exakt (löst nach der Verschiebung entlang der Einwärts-Richtung auf, bei der
x- oder y-Abstand gerade ausreicht), mit `maxPushCap = size*0.25` nur als
Sicherheitsobergrenze für Extremfälle. Zusätzlich läuft die Auflösung
zwischen Zeit-Labels (`resolveLabelCollisions`) und die mit dem Kompass
(`resolveCompassCollisions`) in 3 abwechselnden Runden — ein Ausweichen vor
dem einen kann sonst eine neue Kollision mit dem anderen erzeugen. Bitte bei
künftigen Änderungen an dieser Logik nicht wieder auf ein festes
Iterations-/Schritt-Budget zurückfallen.

---

## Datum/Uhrzeit manuell einstellbar (Nutzer-Feature)

Der Nutzer kann Datum und Uhrzeit direkt anklicken und ändern — dafür werden
bewusst NATIVE Browser-Eingabefelder verwendet (`<input type="date">` /
`<input type="time">`, `#dateInput` / `#timeInput`), keine externe
Picker-Bibliothek. Klick öffnet automatisch den system-/browsereigenen
Kalender- bzw. Uhrzeit-Picker (auf Desktop wie auf dem Smartphone jeweils
mit der nativen UI) — passt zum Projektgrundsatz "keine unzuverlässigen
externen Abhängigkeiten" (vgl. Planeten-Eigenimplementierung,
lokale JSON-Dateien statt CDN).

**Verhalten:** Eine Änderung an einem der Felder (`change`-Event)
setzt `virtualAnchorSim`/`virtualAnchorReal` neu (siehe
`applyDateTimeInputs`) — die virtuelle Zeit "springt" auf den gewählten
Zeitpunkt, das aktuell eingestellte Zeitraffer-Tempo bleibt dabei
UNVERÄNDERT (anders als der NOW-Knopf, der zusätzlich das Tempo
zurücksetzt). Die Felder werden pro Tick mit der laufenden virtuellen Zeit
synchronisiert (`syncDateTimeInputs`), aber NUR wenn keines davon
gerade fokussiert ist — sonst würde der 10×/Sekunde-Tick die Eingabe mitten
in der Bedienung des Pickers überschreiben. Bitte diese Fokus-Prüfung bei
Änderungen an der Sync-Logik beibehalten.

**Historische Daten inkl. v. Chr. (WICHTIG, nicht vereinfachen):** Ein
`<input type="date">` kann prinzipiell KEINE Jahre vor 1 darstellen. Das
Jahr lebt deshalb in einem separaten `#yearInput` (type="number",
**astronomische Jahreszählung**: 0 = 1 v. Chr., −562 = 563 v. Chr.,
Bereich −9999…9999). Der Jahresteil des date-Inputs wird beim Sync auf
1…9999 geklemmt ("Proxy-Jahr") und ist per CSS ausgeblendet
(`::-webkit-datetime-edit-year-field` — nur Chromium; in anderen Browsern
erscheint das Jahr doppelt, harmlos). Wählt der Nutzer im Kalender-Popup
gezielt ein anderes Jahr, gewinnt das Popup (Erkennung über
`lastSyncedDateValue`-Vergleich in `applyDateTimeInputs`), sonst gilt
immer das Jahr-Feld. Drei Fallstricke, die dabei gelöst wurden — bitte
nicht wieder einbauen:
1. `Date.UTC(80, …)` interpretiert Jahre 0–99 als 1900–1999 → Helfer
   `utcMs` mit `setUTCFullYear` benutzen.
2. `Intl.formatToParts` liefert für v.-Chr.-Daten year="563" OHNE
   Unterscheidung zu 563 n. Chr. — nur mit `era: 'short'` in den
   Formatter-Optionen kommt "BC" mit; `getLocalParts` rechnet dann
   astronomisch um (y = 1 − year). Die era-Option NICHT entfernen.
3. date-Input verlangt vierstellige Jahre ("0080-06-15"), sonst wird der
   Wert verworfen und das Feld leert sich.
Genauigkeits-Hinweis: Für Antike gilt — Sterne/Sternbilder gut (Präzession
eingerechnet, aber keine Eigenbewegung: Arcturus & Co. über 2500 Jahre
~1–2° verschoben), Sonne gut, Mond ~2–3° (ΔT unberücksichtigt), Planeten
einige Grad. Eingaben zählen im proleptisch gregorianischen Kalender.

**Layout:** Die Anzeige (`#debug`, enthält Datum/Uhrzeit-Inputs + Standort-
Zeile) sitzt oben links (`top/left`), auf Schmalbildschirmen
(`max-width: 600px`) oben mittig zentriert.

**Standort ist jetzt ebenfalls editierbar** (`#locationInput`, Textfeld statt
reinem Text): Eingabe von entweder
- **Koordinaten** — erkannt per Regex (`parseCoordinates`), Format
  `LAT, LON` als Dezimalzahlen (z. B. `51.22, 12.32`), KEIN Netzwerk-Zugriff
  nötig, funktioniert immer offline.
- **Stadtname/Adresse** — alles andere wird als Ortsname behandelt und per
  **OpenStreetMap Nominatim** (`geocodePlace`, kostenloser öffentlicher
  Geocoding-Dienst, kein API-Key) in Koordinaten umgewandelt.

**Bewusste Ausnahme von "keine externen Laufzeit-Abhängigkeiten":** Dies ist
die EINZIGE Stelle im Projekt, die zur Laufzeit einen externen Dienst
kontaktiert — nur bei aktiver Nutzereingabe (Ortssuche), nicht beim
normalen Laden/Anzeigen der Uhr. Passt trotzdem zum Projektgrundsatz, weil
es kein Dauerhaftes/Wiederholtes Laden ist (anders als der verworfene
CDN-Ansatz für Sternkatalog/astronomy-engine) und es für Geocoding keine
sinnvolle Offline-Alternative gibt.

`LAT`/`LON` sind seit dieser Änderung `let` statt `const` (waren vorher
fest). Eine Standort-Änderung setzt `cachedDateKey`/`cachedMoonTime` zurück
(`onLocationChanged`, früher `invalidateLocationCaches`), damit Sonnen-/
Mond-Auf-/Untergang sofort mit den neuen Koordinaten neu berechnet werden —
der Tages-Cache reagiert sonst nur auf Datumswechsel, nicht auf
Standortwechsel.

Automatische Standort-Ermittlung per GPS/Sensor bleibt bewusst außen vor
(siehe Scope-Grenze, Phase 2/Android).

---

## Zeitzone folgt dem Standort (WICHTIG für das Kernkonzept)

Alle angezeigten Uhrzeiten (Sonnenzeiger-Winkel, Minutenzeiger,
Auf-/Untergangs-Labels, Datum/Uhrzeit-Eingabefelder) laufen in der
**bürgerlichen Zeit des GEWÄHLTEN ORTES**, nicht des Geräts. Nur so gilt
das Kernversprechen "Sonne = Ortszeit auf der Uhr" auch, wenn ein Ort in
einer anderen Zeitzone gewählt wird (z. B. Dakar vom deutschen Browser aus).

**Umsetzung (zweistufig, bewusst so gewählt):**
1. **Lat/Lon → IANA-Zeitzonenname:** `tz-lookup.js` (lokale Datei im
   Projektordner, ~72 KB, kein Netzwerkzugriff) — `tzlookup(LAT, LON)`
   liefert z. B. `"Africa/Dakar"`, auf Ozeanen `"Etc/GMT+X"`.
2. **Zeitpunkt → Ortszeit:** Browser-eigene `Intl.DateTimeFormat` mit
   `timeZone` (inkl. korrekter Sommerzeit- und Sonderregeln wie Indien
   +5:30) — keine eigene Offset-Datenbank nötig.

Zentrale Funktionen: `updateTimezone` (bei jedem Standortwechsel via
`onLocationChanged`), `getLocalParts` (Zeitpunkt → Orts-Wandzeit-Teile,
einmal pro Frame in `tick()` berechnet und durchgereicht),
`wallTimeToDate` (Umkehrung für die Datum/Uhrzeit-Eingabe, iterativ über
den Offset, konvergiert an Sommerzeitgrenzen in 2 Schritten). SunCalc,
Sternzeit (`getLST`) und die ganze Himmelsmathematik arbeiten weiterhin mit
absoluten Zeitpunkten — dort hat sich NICHTS geändert. Der Tages-Cache-
Schlüssel in `updateRiseSetIfNeeded` nutzt das ORTS-Datum.

**Verifiziert:** Dakar (UTC+0) im Juli — Sonne kulminiert ~81° hoch auf der
NORD-Seite (astronomisch korrekt, Deklination 23° > Breite 14,7°), alle
Zeiten in Dakar-Ortszeit. Das war der Anlass für diese Änderung: vorher
liefen die Uhrzeiten in Geräte-Zeitzone, der Himmel aber am gewählten Ort.

---

## Präzession & Südhalbkugel-Mond (Genauigkeits-Fixes)

**Präzession:** Sternkatalog + Sternbild-Linien liegen in J2000-Koordinaten
vor; ohne Korrektur wären die Sterne 2026 um ~0,36° verschoben (~50,3″/Jahr
wachsend). `updatePrecession` (Meeus Kap. 21, Gl. 21.2–21.4, numerisch
gegen Meeus-Beispiel 21.b verifiziert: 0,00″ Abweichung) rechnet die
Koordinaten auf die aktuelle Epoche um. Rohdaten bleiben als `ra0_h`/
`dec0_deg` bzw. `segments` erhalten; präzessiert wird in `ra_h`/`dec_deg`
bzw. `segmentsP`. Neuberechnung nur bei >30 Tagen Abstand der virtuellen
Zeit zur letzten Epoche (funktioniert daher auch im Zeitraffer und bei
Datums-Sprüngen über Jahrzehnte, vorwärts wie rückwärts).

**Mondphasen-Spiegelung:** Auf der Südhalbkugel (`LAT < 0`) wird die
Mondphasen-Darstellung horizontal gespiegelt (zunehmend = links beleuchtet)
— `drawMoonPhase(..., mirrorX)`. Bewusst nur binäre N/S-Spiegelung, keine
kontinuierliche Drehung nach parallaktischem Winkel (stilisierte Anzeige).

---

## NICHT ÄNDERN OHNE RÜCKFRAGE: Mond-Auf-/Untergangszeiten-Suche

**Bekannter, bereits gefixter Bug:** `SunCalc.getMoonTimes(heute)` liefert
Auf- und Untergang für EINEN Kalendertag — das ist bei einem Mond, der z.B.
heute Abend aufgeht und erst morgen früh untergeht, NICHT dasselbe
Sichtbarkeitsintervall. Ein naiver Ansatz ("nimm rise und set vom selben
Kalendertag") führt zu einer falschen, "zusammengerissenen" Bahn.

**Korrekte Lösung (bereits implementiert in `computeMoonRiseSet`):** Sammle
Mond-Ereignisse aus einem Fenster von gestern/heute/morgen ein, sortiere sie
chronologisch, und finde das gerade laufende oder nächste zusammenhängende
Sichtbarkeitsintervall (abhängig davon, ob der Mond gerade jetzt über dem
Horizont steht oder nicht). Bitte diese Logik nicht durch eine simplere
"einfach getMoonTimes() von heute nehmen"-Version ersetzen.

---

## Datenquellen — STANDARD seit Kürzlichem: lokale JSON-Dateien

- **SunCalc** — Sonnen-/Mondposition, Auf-/Untergangszeiten. Wird weiterhin
  vom CDN geladen (cdnjs.cloudflare.com). https://github.com/mourner/suncalc
- **Sternkatalog + Sternbild-Linien** — NUR als Datenquelle (GeoJSON),
  NICHT als Rendering-Engine (d3-celestials eigene Projektions-API wird
  nicht genutzt, komplettes Rendering läuft über eigene Canvas-Pipeline).
  Dateien: `stars.6.json`, `constellations.lines.json`
  **STANDARD: lokale relative Pfade, NICHT die CDN-URL:**
  ```js
  const url = 'stars.6.json';               // NICHT: https://cdn.jsdelivr.net/...
  const url = 'constellations.lines.json';   // NICHT: https://cdn.jsdelivr.net/...
  ```
  Grund: Auf Android war das Laden vom CDN sehr langsam; lokale Dateien im
  selben Verzeichnis wie `index.html` lösten das. Diese beiden Dateien
  müssen im selben Ordner wie `index.html` liegen (sind auf diesem Pi
  bereits vorhanden).
  Quelle der Originaldaten falls neu benötigt: https://github.com/ofrohn/d3-celestial
- Filterung: nur Sterne mit `mag ≤ 4.5` (~750 Sterne, bewusst reduziert für
  klare Lesbarkeit — mehr Sterne wurden als "zu unruhig" empfunden)
- **Planetenpositionen** — selbst implementiert nach Jean Meeus,
  "Astronomical Algorithms" Kap. 33 (Kepler-Gleichung per Newton-Raphson,
  Genauigkeit ~1-2°). KEINE externe Bibliothek (astronomy-engine wurde
  ausprobiert, CDN/API funktionierte nicht zuverlässig, wurde verworfen).
- **Zeitzonen-Zuordnung** — `tz-lookup.js` (lokale Datei, ~72 KB, muss wie
  die JSON-Dateien im selben Ordner wie `index.html` liegen). Quelle:
  https://github.com/darkskyapp/tz-lookup (npm `tz-lookup@6.1.25`).
  Nur die Lat/Lon→Zeitzonenname-Zuordnung; Offsets/Sommerzeit macht der
  Browser (Intl API). Siehe Abschnitt "Zeitzone folgt dem Standort".

---

## Layout — zwei getrennte Radien

- **R_sky** — Himmelskuppel (Sterne, Sternbilder, Planeten, Sonne, Mond,
  Auf-/Untergangs-Labels)
- **R_outer** — äußerer Zifferblatt-Ring (Stundenzahlen + Teilstriche), liegt
  AUSSERHALB der Himmelskuppel, bewusst so verlegt, damit die Himmels-
  darstellung nicht durch Zahlen/Striche verdeckt wird

Aktuelle Werte: `R_sky = (size/2) * 0.81`, `R_outer = (size/2) * 0.94`

---

## Naturalistischer Tag/Nacht-Himmel

Ein Konzept für drei Anwendungen: Die astronomische "Grenzgröße" (limiting
magnitude — schwächste noch sichtbare Helligkeit) wird aus der Sonnenhöhe
berechnet (stufenlose lineare Interpolation zwischen Referenzpunkten,
definiert in `MAGLIMIT_KEYFRAMES`) und mit der Magnitude jedes Objekts
verglichen:

- **Himmelsfarbe** (`SKY_KEYFRAMES`, `drawSkyBackground`): Kontinuierlicher
  Verlauf Taghimmel → goldene Stunde → Dämmerungsstufen → Nacht, plus warmer
  "Glüh"-Kreis um die aktuelle Sonnenposition (räumlich UND zeitlich
  stufenlos, kein Sprung).
- **Sterne:** blenden abhängig von Helligkeit + aktueller Grenzgröße ein/aus
  (erst helle, dann schwache Sterne — Funktion `magnitudeOpacity`).
- **Sternbild-Linien & Planeten:** nutzen dieselbe Grenzgrößen-Logik.

---

## Design-Elemente

- **Zeiger:** Klassische geschwungene "Spaten"-Form (`drawClassicHand`,
  gefüllte Form, nicht nur Linie), helle Bronze-/Messingfarbe `#d3b07f` mit
  dünnem dunklem Rand `rgba(35,24,12,0.55)` für Kontrast auf jedem
  Himmelshintergrund. Stundenzeiger (Sonne) kürzer & breiter, Minutenzeiger
  länger & schlanker.
- **Kompass:** 8 Himmelsrichtungen, deutsche Abkürzungen (N, NO, O, SO, S,
  SW, W, NW). Hauptrichtungen (N/O/S/W) größer/heller, Zwischenrichtungen
  kleiner in derselben Farbe wie die Zwischenzeiten-Zahlen am Zifferblatt
  (`#aeb9c7`) — bewusst einheitliches Design.

---

## Dauerhaftes Feature: Zeitraffer-Bedienelement

Kassettenrekorder-Optik unten rechts, vier Tasten (⏪ ▶ ⏩ NOW):
- `speedUp()` / `speedDown()`: Geschwindigkeitsstufen `[1, 60, 240, 960,
  3840, 14400]`, symmetrisch auch rückwärts (negatives Vorzeichen)
- `speedNormal()` (▶-Taste): setzt NUR das Tempo auf 1× zurück, virtuelle
  Zeit läuft ab dort normal weiter (kein Sprung)
- `speedReset()` (NOW-Taste): springt zur echten aktuellen Uhrzeit UND setzt
  Tempo zurück

**Bleibt dauerhaft im Projekt** — ursprünglich als reines Test-/Demo-Werkzeug
gedacht, das vor einer "fertigen" Version wieder entfernt werden sollte. Das
war ein Missverständnis: Das Zeitraffer-Element ist ein gewolltes, festes
Feature der App und soll NICHT entfernt werden. `tick()` bleibt auf
`getVirtualNow()`, der HTML/CSS-Block `#transport` bleibt bestehen.

---

## Scope-Grenze — WICHTIG

**Diese Arbeitssitzung betrifft ausschließlich die Web-App (Phase 1).**
Die Android-Umwandlung (Phase 2: Kotlin-WebView-Hülle, GPS-Integration) ist
BEWUSST ausgeklammert und wird an anderer Stelle entwickelt (nicht auf
diesem Raspberry Pi — Grund: Android-SDK-Build-Tools und -Emulator haben
keine verlässliche offizielle ARM64-Linux-Unterstützung, recherchiert und
bestätigt). Bitte keine Android-/Kotlin-/Gradle-Dateien in diesem Projekt
anlegen.

**Aktuell NICHT im Scope:**
- DeviceOrientation / Sensor-Kompass — bewusst gestrichen, der Kompass ist
  ein reines Anzeige-Element (aus der Rotation berechnet), kein Sensor-Feature
- GPS/Geolocation — verschoben nach Phase 2, Standort bleibt fest auf
  Zwenkau (51.22°N, 12.32°O) codiert

---

## Arbeitsweise / Erwartungshaltung

- Bei Änderungen an der Kernmathematik (Rotation, Auf-/Untergangs-Labels,
  Grenzgröße-Logik) bitte kurz erklären, was sich ändert und warum, bevor es
  umgesetzt wird — nicht einfach still durchändern
- Bei mathematischen Behauptungen lieber nachrechnen (z.B. mit einem
  kurzen Python-Snippet) als aus dem Gedächtnis zu behaupten
- Ehrlich benennen, wenn ein gewünschtes Feature mit einer bereits
  getroffenen Design-Entscheidung kollidiert, statt es stillschweigend zu
  verbiegen
- Schrittweise vorgehen, ein Feature nach dem anderen, mit testbarem
  Zwischenstand

---

## Bekannte offene Punkte

1. ~~Standort manuell editierbar machen~~ — erledigt (Stadtname/Adresse via
   Nominatim ODER Koordinaten-Eingabe, siehe oben). Automatische Ortung per
   GPS bleibt bewusst Phase 2/Android.
2. Weitere Änderungsideen — werden zu Beginn dieser Sitzung vom Nutzer
   ergänzt, hier ggf. noch nicht erfasst
3. Nach Abschluss dieser Sitzung: Rückkehr zum Linux-Mint-PC-Workflow
   (`/home/goraran/prog/AstroClock`, GitHub-Repo `GorArania/AstroClock`) —
   Änderungen von hier müssen dorthin zurücksynchronisiert werden (git
   pull/push oder manuelles Kopieren), damit GitHub Pages und der lokale
   Stand nicht auseinanderlaufen
