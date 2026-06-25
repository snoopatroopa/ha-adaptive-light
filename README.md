# 💡 HA Adaptive Light

**Home Assistant Blueprint** für bewegungsgesteuerte, lux-geregelte Beleuchtung.

[![HA Blueprint](https://img.shields.io/badge/Home%20Assistant-Blueprint-blue?logo=home-assistant)](https://www.home-assistant.io/)
[![Version](https://img.shields.io/badge/Version-2026.06.24-green)]()
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

Bis zu 5 Zeiträume mit je eigenem Modus (Lux / Konstant / Nachtlicht / Aus).
Zeitraum 1 gilt ab 00:00 Uhr (Standardmodus).

---

## Betriebsmodi & Ablauf

```
Trigger (Bewegung / Schalter / Lux-Drop)
    │
    ├── Branch 1: Zusatz-Trigger → Einschalten → Warten → Ausschalten
    ├── Branch 2: Nachtlicht    → Einschalten → Warten → Nachlauf → Ausschalten
    ├── Branch 3: Aus-Modus     → nichts tun
    ├── Branch 4: Aufräum       → Nachlauf nachholen → Ausschalten (Waisenkinder-Schutz)
    └── Default:  Lux / Konstant
                    → Einschalten → Regelschleife → Nachlauf → Ausschalten
```

---

## Changelog

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
