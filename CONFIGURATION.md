# Configuration Reference

## Current defaults (v1.0.0)

All settings are in-app. There is no external config file in this version. Settings reset to defaults on page reload — persistent settings are on the roadmap (see `FUTURE.md`).

---

## Filter settings

| Setting | Default | Description |
|---|---|---|
| Min AGL (ft) | 500 | Towers below this height are ignored entirely — not parsed, not diffed, not exported |
| Max range (NM) | 70 | Towers beyond this distance from the reference point are ignored |

The 70 NM default matches the radius of the FAA NOTAM search query. If you change the search radius in the FAA tool, update this value to match.

---

## Alert levels

Three levels, each with an AGL threshold and a KML display color. Level 1 is highest severity.

| Level | Default AGL threshold | Default color | Applies to |
|---|---|---|---|
| Level 1 | ≥ 800 ft AGL | Red `#cc0000` | Critical — tallest unlit towers |
| Level 2 | ≥ 600 ft AGL | Amber `#e07000` | Caution |
| Level 3 | ≥ min AGL | Blue `#1a4f8a` | Advisory — everything else above the qualifier |

Level 3 has no AGL input because it automatically covers the range from the Level 2 threshold down to the minimum AGL qualifier.

Color applies to both the KML icon tint in ForeFlight and the AGL cell color in the results table.

---

## Reference point

**Hardcoded in v1.0.0.** To change, edit the `REF` constant near the top of the `<script>` block in `notam-hazard-tracker.html`:

```javascript
const REF = { lat: 37.5617, lon: -85.1450, id: '6I2' };
```

Making this a user-editable field is the highest-priority item in `FUTURE.md`.

---

## NOTAM parser filters

The parser silently excludes entries that are not obstruction towers with unserviceable lighting. Specifically excluded:

- Entries without `U/S` in the text (lighting is operational or unknown status)
- `CRANE` entries
- `STACK` entries  
- `POWER LINE` entries
- `WIND TURBINE` entries

These exclusions are intentional. Wind turbine farms in particular have complex shapes poorly suited to individual-tower plotting. Cranes are temporary and typically in terminal areas.

If operational requirements change, these filters are a single regex in the `parseTowers()` function.
