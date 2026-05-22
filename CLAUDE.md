# Projektkontext – HA Adaptive Light

## Repo
- GitHub: `snoopatroopa/ha-adaptive-light`
- Blueprint-Datei: `sensor-light-lux.yaml`
- Aktuelle Version: **5.3**

---

## Architektur – Blueprint-Struktur

```
Trigger: motion | switch | lux_drop
    │
    ├── Branch 1: ZUSATZ-TRIGGER (switch)
    │     Einschalten → Warten (by_presence / by_switch / by_both) → Ausschalten
    │
    ├── Branch 2: NACHTLICHT (motion + effective_mode == 'night_light')
    │     Aufräumen → Nachtlicht-Lampen an → Warten → Nachlauf → Ausschalten
    │
    ├── Branch 3: AUS (motion + effective_mode == 'off')
    │     → nichts tun
    │
    ├── Branch 4: AUFRÄUM / Waisenkinder-Schutz
    │     Greift wenn: kein Switch-Trigger, keine Präsenz, kein Bypass,
    │     UND mindestens eine Lampe noch an (lamp1 ODER lamp2-5)
    │     → Nachlauf nachholen → doppelt prüfen → Ausschalten
    │
    └── Default: LUX-REGELUNG oder KONSTANT
          Aktivierungs-Check → Einschalten per Lichtkurve →
          Lux-Regelschleife (while Präsenz) → Nachlauf → Bypass-Guard → Ausschalten
```

---

## Wichtige Design-Entscheidungen

| Thema | Entscheidung |
|-------|-------------|
| `mode: restart` | Neue Trigger unterbrechen laufenden Run → Branch 4 fängt Überbleibsel |
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

---

## Variablen-Referenz (wichtigste)

```yaml
effective_mode:   # 'lux' | 'constant' | 'night_light' | 'off'
lamp2_active:     # true wenn light_entity_2 konfiguriert
curve_brightness_l1..l5:  # Interpolierter Startwert aus 5-Punkt-Kurve
init_bri_1..5:    # Aktuelle Helligkeit wenn an, sonst curve_brightness
lux_deviation:    # target_lux - current_lux_val (positiv = zu dunkel)
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

- [ ] Testen ob Branch-4-Fix das "Lampen bleiben an"-Problem vollständig löst
- [ ] Repo auf Public stellen nach erfolgreichem Test
- [ ] HA Community Forum Post wenn stabil
