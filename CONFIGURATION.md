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

---

## Out of Range section

Entries that meet the height filter (≥ min AGL) but fall outside the max range are listed in a collapsed **Out of Range** section below the diff results. These are excluded from the hazard map diff and from the KML export. The section exists for auditability — to confirm the parser saw an entry and made a deliberate range-based exclusion rather than silently dropping it.

Only the **current month** report contributes to the Out of Range section. Previous month out-of-range entries are parsed and discarded — since tower coordinates don't change, anything out of range last month will also be out of range this month and will appear in the current report's section.

---

## Report consistency checks

After parsing, the tool reads the FAA header from each PDF ("NOTAMs within N NM around XXX, Query ran at UTC...") and checks for the following mismatches, displaying an amber warning banner if any are found:

| Check | What it catches |
|---|---|
| Current report airport vs `REF.id` | PDF queried around a different airport than the tool is configured for |
| Current report radius vs max range setting | FAA report radius and tool max range setting differ by more than 1 NM |
| Previous vs current airport | Two PDFs from different reference points being diffed against each other |
| Previous vs current radius | Two PDFs with different query radii |
| Previous vs current query date | Same file loaded into both drop zones |

The tool still runs and produces results when mismatches are detected — the banner is a warning, not a hard stop. Reviewing the banner before acting on results is part of the monthly workflow.

**Important:** keeping the FAA query radius, the tool's max range setting, and the reference airport consistent across both PDFs and the tool configuration is a **user responsibility**. The mismatch detection catches obvious errors but cannot account for all edge cases (e.g., a deliberate radius change mid-month).

---

## Unlit triggers

The parser recognizes two conditions as "unlit":

| Code | Meaning |
|---|---|
| `U/S` | Light exists but is unserviceable (broken) |
| `NOT LGTD` | Structure is not lit — either never was, or lighting was removed |

Both are operationally equivalent from a hazard perspective and are treated identically.

---

## Structure types

| Type | Included | Rationale |
|---|---|---|
| TOWER / TWR | ✓ | Primary obstruction type |
| STACK | ✓ | Tall industrial stacks present the same collision hazard as towers |
| CRANE | ✗ | Temporary, typically in terminal areas, frequently updated |
| POWER LINE | ✗ | Linear feature — single-point plotting is misleading |
| WIND TURBINE FARM | ✗ | Farm-level entry covers a radius area; individual turbines not itemized |

---

## Buffer zone

### Operational rationale

FAR Part 135 and company GOM set minimum terrain/obstacle clearance at 300 ft AGL day and 500 ft AGL night. The hazard map threshold is set at 500 ft to focus on unlit obstacles relevant to night operations.

However, the 500 ft floor is a regulatory minimum, not a guaranteed separation. In practice, pilots maintaining "not lower than 500 ft" in rolling terrain may momentarily dip below that value. Departure and approach phases — where obstacle proximity is most critical — are also high-workload environments where precise altitude control is harder to maintain. A tower at 490 ft AGL is operationally indistinguishable from one at 510 ft.

For these reasons, the tool supports a configurable **buffer zone** below the minimum AGL threshold. Towers within this buffer are parsed and included in the ForeFlight KML overlay for in-cockpit situational awareness, but are **not** included in the hazard map diff (Remove / Keep / Add) and should not be plotted on the wall map. The distinction preserves the clarity of the office hazard map while ensuring the cockpit overlay is more conservative.

### Default values

| Setting | Default | Rationale |
|---|---|---|
| Min AGL | 500 ft | Night obstacle clearance standard (FAR 135 / company GOM) |
| Buffer | 50 ft | Fixed margin accounting for terrain-induced altitude variation; captures towers down to 450 ft AGL |
| Effective KML floor | 450 ft | Min AGL minus buffer |

The buffer is intentionally a fixed value rather than a percentage. The terrain-variation risk that motivates the buffer does not scale with the threshold — a 50 ft margin is appropriate whether the threshold is 500 ft or 800 ft. A percentage-based buffer would over-extend at high thresholds and under-protect at low ones.

### What appears where

| Tower AGL | Wall hazard map | ForeFlight KML |
|---|---|---|
| ≥ Min AGL (500 ft) | ✓ plotted (levels 1–3) | ✓ included (levels 1–3) |
| Buffer zone (450–499 ft) | ✗ not plotted | ✓ included (buffer color) |
| < Buffer floor (< 450 ft) | ✗ ignored | ✗ ignored |
| > Max range | ✗ ignored | ✗ ignored |

### Adjusting the buffer

If operational requirements change — for example, if day operations in low-visibility conditions become a concern — both the Min AGL and the buffer can be adjusted in the Filter & Alert Settings panel. Common configurations:

- **Night standard, conservative buffer:** Min AGL 500 ft, buffer 50 ft → KML floor 450 ft
- **Night standard, tighter buffer:** Min AGL 500 ft, buffer 25 ft → KML floor 475 ft
- **Day standard with buffer:** Min AGL 300 ft, buffer 50 ft → KML floor 250 ft
