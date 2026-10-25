# 💡 HA Adaptive Light

**Home Assistant Blueprint** für bewegungsgesteuerte, lux-geregelte Beleuchtung.

[![HA Blueprint](https://img.shields.io/badge/Home%20Assistant-Blueprint-blue?logo=home-assistant)](https://www.home-assistant.io/)
[![Version](https://img.shields.io/badge/Version-2026.10.25.2-green)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Features

| Feature | Beschreibung |
|---------|-------------|
| 🔆 **Lux-Regelung** | Aktive Regelschleife hält konfigurierten Ziel-Lux-Wert |
| 💡 **Konstant-Modus** | Feste Helligkeit ohne Nachregeln |
| 🌙 **Nachtlicht-Modus** | Separate Helligkeit + Farbe, pro Lampe deaktivierbar |
| ⛔ **Aus-Modus** | Keine Reaktion auf Bewegung |
| 📈 **5-Punkt-Lichtkurve** | Lineare Interpolation der Starthelligkeit je Umgebungslux |
| 🕐 **5 Zeiträume** | Unterschiedliche Modi je Tageszeit konfigurierbar |
| 🌅 **Auto-Nachtriggern** | Licht geht automatisch an wenn Umgebungslux sinkt |
| ☀️ **Auto-Ausschalten** | Licht geht aus wenn Umgebung zu hell wird |
| 🔀 **Zusatz-Trigger** | Optionaler Schalter-Trigger mit eigenem Lifecycle |
| 🎨 **RGB & Farbtemperatur** | Pro Lampe und Modus konfigurierbar |
| 🔓 **Bypass** | `input_boolean` hält Licht dauerhaft aktiv |
| 😴 **Ruhemodus** | Schalter pausiert die Automation – Lampen aus, dauerhaft Nachtlicht oder Nachtlicht bei Bewegung |
| 👥 **Bis zu 5 Lampen** | Individuell kalibrierbar |
| 📡 **Bis zu 5 Sensoren** | Bewegungs- und Präsenzsensoren, inkl. Gruppen |

---

## Installation

### Option A – Import-Link (empfohlen)

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/snoopatroopa/ha-adaptive-light/blob/main/sensor-light-lux.yaml)

### Option B – Manuell

1. Datei [`sensor-light-lux.yaml`](sensor-light-lux.yaml) herunterladen
2. In HA-Konfigurationsordner kopieren: `config/blueprints/automation/`
3. Home Assistant neu starten oder Blueprints neu laden

---

## Konfiguration

### Pflichtfelder

| Feld | Beschreibung |
|------|-------------|
| Bewegungssensoren | Mindestens 1 Sensor (binary_sensor oder group) |
| Licht #1 | Hauptlampe (muss `brightness` unterstützen) |
| Lux-Sensor | Sensor mit `device_class: illuminance` |

### Lichtkurve kalibrieren

1. Licht ausschalten, Umgebungslux ablesen → Wert in **Lux-Messpunkt** eintragen
2. Lampen manuell dimmen bis Ziel-Lux erreicht → Helligkeit in **Lichtkurve** eintragen
3. Für alle 5 Punkte wiederholen (von dunkel → hell)

### Zeiträume

Bis zu 5 Zeiträume mit je eigenem Modus (Lux / Konstant / Nachtlicht / Aus),
eigenem Ziel-Lux und eigener Konstant-Helligkeit (0 = Werte der einzelnen Lampen).
Zeitraum 1 gilt ab 00:00 Uhr (Standardmodus).

---

## Betriebsmodi & Ablauf

```
Trigger (Bewegung an / Bewegung aus / Schalter / Lux-Drop / Ruhemodus an/aus)
    │
    ├── Branch 0: Ruhemodus     → alle Trigger ignorieren; beim Aktivieren Lampen aus
    │                             (Aktion "Nachtlicht" → weiter in Branch 3 ohne Ausschalten;
    │                              "Nachtlicht bei Bewegung" → Nachtlicht-Modus erzwungen,
    │                              nur Zusatz-Trigger geblockt)
    ├── Branch 1: Zusatz-Trigger → Einschalten → Warten → Ausschalten
    ├── Branch 2: Bewegung-Aus  → Nachlauf → erneute Prüfung → Ausschalten
    ├── Branch 3: Nachtlicht    → Direkt-Einschalten (Nachtwerte) → Warten → Nachlauf → Ausschalten
    ├── Branch 4: Aus-Modus     → nichts tun
    ├── Branch 5: Aufräum       → Nachlauf nachholen → Ausschalten (Waisenkinder-Schutz)
    └── Default:  Lux / Konstant (Zeitraum-Modus per Modus-Override übersteuerbar, außer Nachtlicht)
                    → Einschalten → Regelschleife → Nachlauf → Ausschalten
```

---

## Changelog

### 2026.10.25.2
- **Konstant-Helligkeit pro Zeitraum.** Neue Einstellung `period_N_constant_bri`
  (0–100 %) im Reiter „Zeiträume & Betriebsmodi“. Steht der Zeitraum – oder der
  Modus-Override – auf „Konstante Helligkeit“, gehen alle Lampen auf diesen Wert.
  0 (Standard) = wie bisher die Konstant-Helligkeit der einzelnen Lampen.

### 2026.10.25
- **Modus-Override: wählbare übersteuerte Modi.** Neue Mehrfachauswahl
  `mode_override_modes` (Aus / Lux-Regelung / Konstant; Standard: nur Aus =
  bisheriges Verhalten). Beispiel: Schalter = „Rollo nicht offen“, Ziel-Modus
  Konstant, übersteuert nur Lux → bei geschlossenem Rollo wird nicht nach Lux
  geregelt, sondern konstant eingeschaltet. Nachtlicht und Ruhemodus haben
  immer Vorrang und werden nie übersteuert. Nachtlicht-Zeiträume bei
  deaktivierter Nachtlicht-Funktion gelten als „Aus“ (vorher griff der
  Override dort nicht).

### 2026.10.19.2
- **Ruhemodus: dritte Aktion „🚶 Nachtlicht bei Bewegung“.** Solange der Ruhemodus
  an ist, wird unabhängig vom Zeitraum der Nachtlicht-Modus erzwungen: Bewegung
  schaltet das Nachtlicht ein, ohne Anwesenheit geht es nach `night_time_delay`
  wieder aus (regulärer Bewegung-Aus-Branch). Beim Aktivieren mit Anwesenheit
  sofort Nachtlicht, ohne Anwesenheit sofort aus. Lux-Regelung, Konstant und
  Zusatz-Trigger bleiben bis zum Ausschalten des Ruhemodus gesperrt.

### 2026.10.19
- **Neues Feature: Ruhemodus.** Optionaler `input_boolean`- oder `binary_sensor`-Schalter
  (`rest_mode_switch`) plus Aktion (`rest_mode_action`: alle Lampen aus oder
  dauerhaft Nachtlicht). Solange der Schalter an ist, ignoriert die Automation
  alle anderen Trigger. Für das Nachtlicht werden die bestehenden
  Nachtlicht-Einstellungen pro Lampe verwendet (keine zweite Konfiguration).
  Beim Ausschalten wird bei Anwesenheit sofort der aktuell gültige Modus
  angewendet; die Lux-Regelung startet dabei mit der Lichtkurve und dem echten
  Lux-Aktivierungscheck statt mit der Nachtlicht-Helligkeit. Ohne Anwesenheit
  gehen noch leuchtende Lampen nach der Nachlaufzeit aus (Aufräum-Branch).

### 2026.08.16
- **Neues Feature: Modus-Override.** Optionaler `input_boolean`- oder `binary_sensor`-Schalter
  (`mode_override_switch`) plus Ziel-Modus-Auswahl (`mode_override_mode`:
  Lux-Regelung oder Konstante Helligkeit). Steht der aktuell aktive
  Zeitraum auf "⛔ Aus" und der Schalter ist an, läuft bei Bewegung
  stattdessen der gewählte Modus – z.B. um die Lux-Regelung außerhalb der
  sonst konfigurierten Zeiten einmalig zu aktivieren. Zeiträume, die
  bereits auf Lux/Konstant stehen, bleiben unverändert; Nachtlicht behält
  immer Vorrang und wird nie überschrieben. Unabhängig vom bestehenden
  Bypass-Schalter nutzbar (der weiterhin nur das automatische Ausschalten
  unterdrückt) – beide lassen sich kombinieren.

### 2026.07.21
- **Bugfix:** Nachtlicht-Branch schaltete beim Einschalten grundsätzlich
  zuerst alle Lampen aus, wartete pauschal `light_transition_off + 0.5s` und
  schaltete die Nachtlicht-Lampen danach separat wieder ein. Bei Geräten mit
  trägerer Aus-Rückmeldung (beobachtet: ~2,6s realer Latenz gegenüber 2,5s
  Wartezeit) traf die verspätete Aus-Bestätigung erst nach dem neuen
  Nachtlicht-`turn_on` ein und überschrieb die gerade gesetzte Farbe.
  Reproduziert auf `automation.lichtsteuerung_bad_og_bei_bewegung`
  (`light.spiegellampe_cob_strip`) per Logbuch: on → off nach 68ms → on
  nach 30s (letzteres ohne Automations-Kontext).
  Fix: Lampen, die im Nachtlicht-Modus aktiv bleiben, werden direkt per
  einzelnem `turn_on` auf die Nachtlicht-Werte gesetzt (kein
  Ausschalten-vor-Einschalten-Umweg mehr, keine Wartezeit an
  Geräte-Latenz gekoppelt). Nur für das Nachtlicht deaktivierte oder nicht
  konfigurierte Lampen werden weiterhin ausgeschaltet.

### 2026.07.20
- **Bugfix:** Der 2026.07.13-Fix (Helligkeit/Transition/Farbe in einem
  einzigen `turn_on`-Aufruf) war nur im Nachtlicht-Branch umgesetzt. Der
  `default`-Zweig (Lux-Regelung & Konstant) hatte beim initialen Einschalten
  weiterhin zwei separate `light.turn_on`-Aufrufe (zuerst `brightness_pct` +
  `transition`, ~1 ms später `rgb_color` / `color_temp_kelvin`). Bei
  RGBCCT-Lampen konnte der zweite Aufruf mitten in die vom ersten Aufruf
  gestartete Transition fallen und wurde ignoriert – die Hauptfarbe wurde
  nicht übernommen. Per Automations-Trace auf Südendamm50 reproduziert.
  Fix: alle 5 Lampen im `default`-Zweig senden jetzt ebenfalls Helligkeit,
  Transition und Farbe in einem einzigen Aufruf.

### 2026.07.13
- **Bugfix:** Nachtlicht-Branch sendete Helligkeit und Farbe in zwei separaten
  `light.turn_on`-Aufrufen (zuerst `brightness_pct` + `transition`, dann 2 ms
  später `rgb_color` / `color_temp_kelvin`). Lampen die sowohl `color_temp` als
  auch `rgb` unterstützen (RGBCCT), starteten beim ersten Aufruf eine Transition
  mit der zuletzt verwendeten Farbtemperatur und ignorierten den nachfolgenden
  Farb-Befehl, der mitten in die laufende Transition ankam – die konfigurierte
  Nachtlicht-RGB-Farbe wurde nicht übernommen.
  Fix: Helligkeit, Transition und Farbe werden jetzt in einem einzigen
  `turn_on`-Aufruf pro Lampe übergeben.

### 2026.07.10
- **Bugfix:** Ausschalten der Lampen war bisher kein eigenständiger Trigger,
  sondern nur Teil des laufenden Einschalt-Durchlaufs. Bei `mode: restart`
  konnte ein neuer Durchlauf (z. B. erneute Bewegung, während inzwischen die
  Aktivierungsbedingung gekippt war – etwa weil der Umgebungslux durch das
  eigene Lampenlicht über die Schwelle stieg) den wartenden Durchlauf
  abbrechen. Der neue Durchlauf tat dann nichts und beendete sich sofort –
  damit wartete kein Durchlauf mehr auf „Bewegung endet", die Lampen blieben
  dauerhaft an.
  Fix: Neuer eigenständiger `state`-Trigger „Bewegung → aus" schaltet jetzt
  unabhängig vom Zustand jedes anderen Durchlaufs zuverlässig aus (inkl.
  Nachlauf und erneuter Präsenz-/Bypass-/Schalter-Prüfung). Zusätzlich prüft
  die Lux-Regelschleife die Bewegung jetzt direkt vor jedem Stellschritt
  erneut, damit kein Stellschritt (inkl. eines möglichen Wieder-Einschaltens)
  kurz nach dem Ausschalten mehr feuern kann.

### 2026.06.24
- **Bugfix:** Nachtlicht-Branch wartete nach dem Aufräumen (Ausschalten aller
  Lampen) nur fest 500 ms, bevor die Lampen mit Nachtlicht-Helligkeit/-Farbe
  wieder eingeschaltet wurden. War die konfigurierte Ausschalt-Übergangszeit
  (`light_transition_off`, Default 2 s) länger als 500 ms, lief die Lampe noch
  mitten in der Ausschalt-Transition und übernahm teils die alte Hauptfarbe
  statt der Nachtlicht-RGB-Farbe.
  Fix: Wartezeit richtet sich jetzt nach `light_transition_off` statt einem
  festen Wert.

### 2026.06.21
- **Bugfix:** Race Condition im Lux-Regelkreis. Der Lux-Step-Up feuerte noch
  im letzten Schleifendurchlauf, nachdem der Bewegungssensor bereits
  abgefallen war. Geräte mit sequenzieller Transition-Verarbeitung (DALI /
  bestimmte Zigbee-Firmware) schalteten sich dadurch 1–2 Sekunden nach dem
  Ausschalten erneut ein.
  Fix: Lux-Messblock prüft nun zusätzlich ob der Bewegungssensor noch aktiv
  ist, bevor ein Step-Up ausgelöst wird.
- **Neu:** Sechster Zeitraum (Zeitraum 6). Jeder Zeitraum erhält ein eigenes
  Ziel-Lux-Feld (`Ziel-Lux pro Zeitraum`). Ist der Wert 0, wird das globale
  Ziel-Lux aus dem Bereich „Lux-Regelung" als Fallback verwendet. Bestehende
  Instanzen müssen nicht angepasst werden.

### v5.3
- **Bugfix:** Aufräum-Branch (Branch 4) prüfte bisher nur ob Lampe 1 an ist.
  Wenn Lampe 1 durch Auto-Off oder einen unterbrochenen Ausschalt-Zyklus bereits
  aus war, blieben Lampen 2–5 dauerhaft an.
  Fix: Branch 4 greift jetzt wenn **irgendeine** konfigurierte Lampe noch leuchtet.

### v5.2
- RGB-Farben pro Lampe und Modus
- Nachtlicht-Farbe unabhängig von Hauptfarbe
- 5 Zeiträume (statt 2)
- Auto-Nachtriggern bei sinkendem Umgebungslicht
- Auto-Ausschalten bei zu hellem Umgebungslicht
- 2-Sample Lux-Mittelung gegen Sensorrauschen

---

## Anforderungen

- Home Assistant 2024.x oder neuer
- Dimmbare Lampen (Zigbee, Z-Wave, WLAN etc.) mit `brightness`-Attribut
- Lux-Sensor mit `device_class: illuminance`

---

## Lizenz

MIT – siehe [LICENSE](LICENSE)
