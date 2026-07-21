# Projektkontext – HA Adaptive Light

## Sprache
- Immer auf Deutsch antworten (Chat-Antworten), unabhängig von der Sprache der Anfrage.

## Repo
- GitHub: `snoopatroopa/ha-adaptive-light`
- Blueprint-Datei: `sensor-light-lux.yaml`
- Aktuelle Version: **2026.07.21** (Repo nutzt Datumsformat statt vX.Y, siehe Version-Badge in README.md)

---

## Architektur – Blueprint-Struktur

```
Trigger: motion | motion_off | switch | lux_drop
    │
    ├── Branch 1: ZUSATZ-TRIGGER (switch)
    │     Einschalten → Warten (by_presence / by_switch / by_both) → Ausschalten
    │
    ├── Branch 2: BEWEGUNG-AUS (motion_off – eigenständiger Aus-Trigger)
    │     Nachlauf (time_delay / night_time_delay) → Präsenz+Bypass+Switch
    │     erneut prüfen → alle konfigurierten Lampen aus
    │     (Vorrang für Branch 1: schaltet nicht aus wenn Zusatz-Trigger aktiv an ist)
    │
    ├── Branch 3: NACHTLICHT (motion + effective_mode == 'night_light')
    │     Direkt-Einschalten auf Nachtwerte (kein Aus-vor-An mehr, seit
    │     2026.07.21) → Warten → Nachlauf → Ausschalten
    │
    ├── Branch 4: AUS (motion + effective_mode == 'off')
    │     → nichts tun
    │
    ├── Branch 5: AUFRÄUM / Waisenkinder-Schutz (Sicherheitsnetz für Randfälle,
    │     z.B. lux_drop-Trigger; regulärer Aus-Fall läuft seit 2026.07.10 über Branch 2)
    │     Greift wenn: kein Switch-Trigger, keine Präsenz, kein Bypass,
    │     UND mindestens eine Lampe noch an (lamp1 ODER lamp2-5)
    │     → Nachlauf nachholen → doppelt prüfen → Ausschalten
    │
    └── Default: LUX-REGELUNG oder KONSTANT
          Aktivierungs-Check → Einschalten per Lichtkurve →
          Lux-Regelschleife (while Präsenz, Bewegung vor jedem Stellschritt
          erneut geprüft) → Nachlauf → Bypass-Guard → Ausschalten
```

---

## Wichtige Design-Entscheidungen

| Thema | Entscheidung |
|-------|-------------|
| `mode: restart` | Neue Trigger unterbrechen laufenden Run → eigener `motion_off`-Trigger (Branch 2, seit 2026.07.10) garantiert trotzdem ein Ausschalten; Branch 5 fängt verbleibende Randfälle |
| Lichtkurve | 5-Punkt lineare Interpolation, Template unvermeidbar |
| Lux-Messung | 2-Sample Mittelung (3s Delay) gegen Sensorrauschen |
| Auto-Off | Nur wenn lamp1 ≤ auto_off_brightness UND zu hell → alle Lampen aus |
| Farben | RGB überschreibt Farbtemperatur; Nachtlicht-Farbe unabhängig von Hauptfarbe |
| `lamp2_active` | `{{ light_entity_2 | length > 0 }}` – prüft ob Entität konfiguriert |

---

## HA Best Practices (immer einhalten)

- `action:` statt `service:` (deprecated)
- `trigger: state` statt `platform: state`
- `expand()` für Sensorlisten (unterstützt `group`-Domain)
- `has_value()` Guard vor Lux-Lesungen
- Native `condition: or/and/not` statt Template-Booleans wo möglich
- `condition: numeric_state` / `condition: state` statt Jinja2-Vergleiche
- Templates nur wo keine native Alternative existiert (z.B. Lichtkurven-Interpolation)

---

## Bekannte Bugs & Fixes

### v5.2 – Branch-4-Trigger
**Problem:** `lux_drop`-Trigger mit `mode:restart` konnte laufenden Nachlauf
abbrechen → neuer Run fand keine Präsenz → Default übersprang → Lampen blieben an.
**Fix:** Branch 4 als Aufräum-Branch eingeführt.

### v5.3 – Branch-4-Lampen-Check
**Problem:** Branch 4 prüfte nur `light_entity_1 state: on`. Wenn lamp1 durch
Auto-Off oder unterbrochenen Ausschalt-Zyklus bereits aus war, blieben lamp2–5
dauerhaft an.
**Fix:** `condition: or` prüft jetzt alle konfigurierten Lampen.

### 2026.07.10 – Kein eigenständiger Aus-Trigger
**Problem:** Ausschalten war bisher kein eigenständiger Trigger, sondern nur
Teil des laufenden Einschalt-Durchlaufs (Regelschleife/Warte-Logik wartet
intern auf Bewegungsende). Mit `mode: restart` konnte ein neuer Durchlauf
(z.B. erneute Bewegung) den wartenden Durchlauf abbrechen. Kippte dabei die
Aktivierungsbedingung (typischerweise Morgendämmerung: Lux steigt – teils
durch das eigene Lampenlicht – über `activation_lux_threshold`), nahm der
neue Durchlauf den `default`-Zweig ohne Aktion und beendete sich sofort.
Damit existierte kein Durchlauf mehr, der auf „Bewegung aus" wartete –
die Lampen blieben dauerhaft an, auch wenn die Präsenz danach mehrfach
auf `off` wechselte.
**Fix:** Neuer eigenständiger `state`-Trigger `motion → off` (`id:
motion_off`, Branch 2) schaltet unabhängig vom Zustand jedes anderen
Durchlaufs zuverlässig aus, inkl. Nachlauf und erneuter Präsenz-/Bypass-/
Switch-Prüfung gegen restart-Rennen. Zusätzlich prüft die Lux-Regelschleife
die Bewegung jetzt direkt vor jedem Stellschritt erneut (nicht mehr nur am
Schleifenanfang), damit kein Stellschritt – inkl. eines möglichen
Wieder-Einschaltens – kurz nach Bewegungsende mehr feuern kann.

### 2026.07.13 – Zwei-Aufruf-Farbrace im Nachtlicht-Branch
**Problem:** Helligkeit/Transition und Farbe wurden pro Lampe in zwei
getrennten `light.turn_on`-Aufrufen gesendet. Bei RGBCCT-Lampen startete
der erste Aufruf eine Transition mit der zuletzt aktiven Farbe, der ~2ms
später folgende Farb-Aufruf kam mitten in die laufende Transition und
wurde ignoriert.
**Fix:** Helligkeit, Transition und Farbe in einem einzigen
`turn_on`-Aufruf pro Lampe (nur im Nachtlicht-Branch umgesetzt).

### 2026.07.20 – derselbe Zwei-Aufruf-Fehler im `default`-Zweig
**Problem:** Der 2026.07.13-Fix wurde nur im Nachtlicht-Branch eingebaut.
Der `default`-Zweig (Lux-Regelung & Konstant) hatte beim initialen
Einschalten denselben Zwei-Aufruf-Fehler weiterhin – per Trace auf
`automation.lichtsteuerung_bad_og_bei_bewegung` nachgewiesen (Aufrufe
1,2ms auseinander).
**Fix:** Alle 5 Lampen im `default`-Zweig senden jetzt ebenfalls
Helligkeit, Transition und Farbe in einem Aufruf (analog Nachtlicht).

### 2026.07.21 – Aus-vor-An-Umweg im Nachtlicht-Branch kollidiert mit trägen Geräten
**Problem:** Der Nachtlicht-Branch schaltete beim Einschalten grundsätzlich
zuerst alle Lampen aus, wartete `light_transition_off + 0.5s` und schaltete
die aktiven Nachtlicht-Lampen danach separat wieder ein. Bei Geräten mit
trägerer Aus-Rückmeldung (beobachtet: ~2,6s realer Latenz gegenüber 2,5s
Wartezeit) traf die verspätete Aus-Bestätigung des Geräts nach dem neuen
Nachtlicht-`turn_on` ein und überschrieb die gerade gesetzte Farbe – die
Lampe meldete sich ~30s später selbständig mit undefiniertem Farbzustand
zurück. Reproduziert auf `automation.lichtsteuerung_bad_og_bei_bewegung`
(`light.spiegellampe_cob_strip`) per Logbuch (on → off nach 68ms → on nach
30s ohne Automations-Kontext). Gleiche Fehlerklasse wie 2026.06.21/06.24,
nur unterhalb der dortigen Toleranz.
**Fix:** Lampen, die im Nachtlicht-Modus aktiv bleiben, werden direkt per
einzelnem `turn_on` auf die Nachtwerte gesetzt – kein Aus-vor-An-Umweg,
keine Wartezeit mehr an Geräte-Latenz gekoppelt. Nur für das Nachtlicht
deaktivierte oder nicht konfigurierte Lampen werden weiterhin per
`turn_off` ausgeschaltet.

---

## Variablen-Referenz (wichtigste)

```yaml
effective_mode:   # 'lux' | 'constant' | 'night_light' | 'off'
lamp2_active:     # true wenn light_entity_2 konfiguriert
curve_brightness_l1..l5:  # Interpolierter Startwert aus 5-Punkt-Kurve
init_bri_1..5:    # Aktuelle Helligkeit wenn an, sonst curve_brightness
lux_deviation:    # target_lux - current_lux_val (positiv = zu dunkel)
off_delay_minutes: # Branch 2 (BEWEGUNG-AUS): night_time_delay falls
                  # effective_mode == 'night_light', sonst time_delay
```

---

## Validierung

Vor jedem Commit YAML validieren:

```python
import yaml

def input_constructor(loader, tag_suffix, node):
    return f"!input {loader.construct_scalar(node)}"

yaml.add_multi_constructor('!', input_constructor, Loader=yaml.SafeLoader)

with open('sensor-light-lux.yaml') as f:
    yaml.safe_load(f.read())
```

---

## Offene Punkte / Ideen

- [ ] Testen ob der 2026.07.10-Fix (eigenständiger `motion_off`-Trigger) das
      "Lampen bleiben an"-Problem auf Südendamm50 (`automation.lichtsteuerung_bad_og_bei_bewegung`)
      vollständig löst – siehe Abnahmekriterien im Fehlerbericht vom 10.07.2026
- [ ] Repo auf Public stellen nach erfolgreichem Test
- [ ] HA Community Forum Post wenn stabil

---

## Changelog

### 2026.07.10 – Eigenständiger Aus-Trigger (Branch 2: BEWEGUNG-AUS)
**Änderung:** Neuer `state`-Trigger `motion → off` (`id: motion_off`) plus
neuer Branch 2, der unabhängig vom Zustand jedes laufenden Durchlaufs
ausschaltet (Nachlauf + erneute Präsenz-/Bypass-/Switch-Prüfung). Lux-
Regelschleife prüft Bewegung jetzt zusätzlich direkt vor jedem Stellschritt.
**Grund:** Ausschalten hing bisher am Überleben eines laufenden
Einschalt-Durchlaufs; `mode: restart` konnte diesen Durchlauf jederzeit
abbrechen und damit jeden zukünftigen Ausschalt-Pfad verlieren. Details
siehe „Bekannte Bugs & Fixes – 2026.07.10" oben.

### v5.3.1 – Branch-4-Condition nativ
**Änderung:** `"{{ trigger.id != 'switch' }}"` in Branch 4 durch native
`condition: not / condition: trigger id: switch` ersetzt.
**Grund:** Wird bei HA-Load validiert statt erst zur Laufzeit (HA Skill Best Practice).
