# AstroClock

Eine astronomisch korrekte 24-Stunden-Uhr als Web-App: **Die Sonne ist der
Stundenzeiger**, und der komplette Sternenhimmel dreht sich passend dazu mit.

**Live:** https://gorarania.github.io/AstroClock/

## Das Konzept

Ein festes 24-Stunden-Zifferblatt (0/24 oben, 6 rechts, 12 unten, 18 links)
zeigt die bürgerliche Ortszeit. Die Sonne sitzt als Stundenzeiger immer exakt
auf ihrer Uhrzeit-Marke; ihr Abstand vom Zentrum zeigt die Höhe über dem
Horizont (Mitte = Zenit, Rand = Horizont). Der Sternhintergrund — Sterne,
Sternbilder, Mond und Planeten — rotiert so, dass alle Himmelsobjekte an
ihrer echten Himmelsrichtung stehen.

Eine verifizierte Eigenschaft dieser Konstruktion: Legt man die Uhr flach hin
und richtet den Sonnenzeiger auf die echte Sonne aus, zeigt der N-Marker nach
geografisch Norden.

## Funktionen

- **Naturalistischer Himmel:** stufenloser Übergang Tag → goldene Stunde →
  Dämmerung → Nacht; Sterne blenden nach Helligkeit ein und aus
- **~900 Sterne** (bis 4,5 mag) und 89 Sternbilder, mit Präzession auf die
  eingestellte Epoche gerechnet
- **Sonne, Mond (mit Phase) und fünf Planeten**, vollständig selbst
  implementiert nach Jean Meeus, *Astronomical Algorithms* (inkl.
  ΔT-Korrektur der Erdrotation)
- **Auf-/Untergangszeiten** von Sonne und Mond als Labels an der echten
  Himmelsrichtung
- **Ort frei wählbar:** Stadtname/Adresse (via OpenStreetMap Nominatim) oder
  Koordinaten; alle Uhrzeiten laufen in der Zeitzone des gewählten Ortes
- **Zeit frei wählbar:** von 9999 v. Chr. bis 9999 n. Chr. — die Jahreszeiten
  stimmen dank eigener Sonnentheorie auch über Jahrtausende; der Himmel zu
  Buddhas Zeiten ist genauso abrufbar wie die Sonnenfinsternis vom 12.8.2026
- **Zeitraffer:** vor- und rückwärts bis 14400×, im Kassettenrekorder-Look

## Technik

Eine einzige `index.html` (Canvas 2D, Vanilla JS) plus lokale Datendateien —
**keine CDN- oder Laufzeit-Abhängigkeiten** (einzige Ausnahme: die optionale
Ortssuche per Nominatim):

| Datei | Inhalt |
|---|---|
| `index.html` | komplette App (Rendering, Astronomie, UI) |
| `stars.6.json` | Sternkatalog (GeoJSON, aus [d3-celestial](https://github.com/ofrohn/d3-celestial)) |
| `constellations.lines.json` | Sternbild-Linien (GeoJSON, ebenfalls d3-celestial) |
| `tz-lookup.js` | Koordinaten → IANA-Zeitzone ([tz-lookup](https://github.com/darkskyapp/tz-lookup)) |

Genauigkeit: Sonne wenige Bogenminuten (heute) bzw. Stunden-genaue
Jahreszeiten über ±5000 Jahre; Mond ~0,2° (heute) / ~1° (Antike);
Planeten ~1–2°; Sternpositionen mit Präzession (Meeus Kap. 21), ohne
Eigenbewegung.

Die Datei `CLAUDE.md` dokumentiert die Design-Entscheidungen im Detail.

## Ausblick (Phase 2)

Verpackung als Android-App (WebView) mit GPS-Ortung — wird separat
entwickelt, dieses Repo enthält die Web-App (Phase 1). Die historischen
Zwischenschritte der Entwicklung liegen unter `phase1/`.
