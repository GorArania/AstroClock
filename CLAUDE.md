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
`drawRiseSetLabel` (ersetzen die früheren `computeSunPath`, `computeMoonPath`,
`drawPath`, `drawPathEndpointLabel`). Bitte hier nicht wieder eine
Bahnlinie einführen, ohne das explizit abzusprechen — das war ein bewusster
Rückbau auf Nutzerwunsch, kein Zwischenstand.

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

1. Weitere Änderungsideen — werden zu Beginn dieser Sitzung vom Nutzer
   ergänzt, hier ggf. noch nicht erfasst
2. Nach Abschluss dieser Sitzung: Rückkehr zum Linux-Mint-PC-Workflow
   (`/home/goraran/prog/AstroClock`, GitHub-Repo `GorArania/AstroClock`) —
   Änderungen von hier müssen dorthin zurücksynchronisiert werden (git
   pull/push oder manuelles Kopieren), damit GitHub Pages und der lokale
   Stand nicht auseinanderlaufen
