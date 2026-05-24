# Changelog

All notable changes to this project are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

Versions are tagged in Git as `v{MAJOR}.{MINOR}.{PATCH}`:
- **MAJOR** — breaking change to output format or core logic
- **MINOR** — new feature, new export type, new filter option
- **PATCH** — bug fix, UI tweak, documentation update

---

## [1.0.0] — 2026-05-23

Initial working version.

### Added
- PDF drag-and-drop input for current and previous month reports
- FAA NOTAM obstruction parsing via PDF.js (client-side, no server required)
- Filtering: minimum AGL threshold, maximum range from reference point
- Diff logic: Remove / Keep / Add categorization keyed on NOTAM ID
- Three-level configurable alert thresholds with color pickers
- Results table with color-coded AGL column matching alert levels
- CSV export with Action column (REMOVE / KEEP / ADD)
- KML export for ForeFlight — all current qualifying towers, colored by alert level
- KML uses standard Google Earth caution icon; altitude in meters MSL per KML spec
- Light theme UI with collapsible settings panel
- Reference point hardcoded to Lebanon Muni (6I2): 37.5617°N, 85.1450°W

### Known limitations
- Reference airport and coordinates are hardcoded (see `FUTURE.md`)
- NOTAM coordinate parser handles standard 6-digit DMS format only; malformed or non-standard coordinates are silently skipped
- PDF.js loaded from CDN — requires internet on first load; cached thereafter
- No persistent settings storage; threshold preferences reset on page reload

---

## [1.1.0] — 2026-05-24

### Added
- `NOT LGTD` recognized as a second unlit trigger alongside `U/S` — catches obstructions that are permanently unlit rather than having a broken light
- Stacks included in structure type filter (previously excluded); cranes, power lines, and wind turbine farms remain excluded
- Out of Range section in results — lists all entries that met the height filter but fell outside the max range setting; collapsed by default; provides auditability without cluttering the primary output
- Type and Unlit reason columns added to results display and CSV export
- Report configuration mismatch detection — after parsing, compares reference airport and radius from each PDF header against each other and against tool settings; displays amber warning banner if discrepancies are found. Checks: current report airport vs configured REF, current report radius vs max range setting, previous vs current airport, previous vs current radius, same query date on both files (possible duplicate load)
- Help modal — "? How to use" button in header opens a full workflow tutorial covering all six monthly steps, the statute-vs-nautical-miles FAA search radius note, ForeFlight KML import path, and configuration gotchas. Closes on ✕, click-outside, or Escape key

### Changed
- `parseTowers()` now returns `{ results, outOfRange }` object instead of a plain array
- CSV export adds `Type` and `Unlit reason` columns

### Fixed
- Stacks with `NOT LGTD` status (e.g. MKL 05/228) were previously invisible to the parser on two counts: excluded structure type and unrecognized unlit trigger. Both corrected.

### Known limitations
- Mismatch detection depends on the FAA PDF header line "NOTAMs within N NM around XXX" — if the FAA changes their header format, detection will silently degrade rather than false-positive
- Alert level settings and filter preferences still reset to defaults on page reload (persistent settings planned)

---

## [1.1.1] — 2026-05-24

### Fixed
- Altitude parser now handles entries where MSL is reported as `UNKNOWN` (e.g. `UNKNOWN (650FT AGL)`). Previously the parser required a numeric MSL value before the AGL parenthetical and silently dropped these entries — they now parse correctly and appear in results or the Out of Range section as appropriate. MSL displays as `—` in the table when not reported.
- Radius mismatch detection tolerance tightened from ±5 NM to ±1 NM. The ±5 window was based on a mistaken assumption that the FAA search radius uses statute miles requiring conversion. The FAA PDF header reports the radius in NM directly, so a tight tolerance is correct.
- Help modal and CONFIGURATION.md corrected to remove inaccurate statute-miles note. The FAA search input uses miles but the PDF header states radius in NM.

---

## [1.2.0] — 2026-05-24

### Added
- Buffer zone below min AGL threshold — configurable in Filter & Alert Settings, default 50 ft. Towers within the buffer (default: 450–499 ft AGL) are:
  - Parsed and included in the ForeFlight KML overlay with a distinct color (default gray)
  - Listed in a **Buffer Zone** section in the results (collapsed by default)
  - Labeled `BUFFER` in CSV export
  - Not included in the hazard map diff (Remove / Keep / Add) — for cockpit awareness only
- Buffer color picker (Level 4) added to KML Alert Levels settings
- Summary bar shows buffer entry count when buffer entries are present
- KML legend updated to show buffer color and floor altitude

### Documentation
- CONFIGURATION.md: full operational rationale for the buffer zone, including FAR 135 / GOM minimums, terrain-variation reasoning, fixed-vs-percentage buffer analysis, and a table showing what appears where at each altitude band

### Design rationale
The buffer is a fixed value (not percentage-based) because terrain-induced altitude variation risk does not scale with the operational threshold. A fixed 50 ft margin is appropriate across all likely threshold configurations.

### Changed
- `parseTowers()` now returns `{ results, outOfRange }` object instead of a plain array
- CSV export adds `Type` and `Unlit reason` columns

### Fixed
- Stacks with `NOT LGTD` status (e.g. MKL 05/228) were previously invisible to the parser on two counts: excluded structure type and unrecognized unlit trigger. Both corrected.

### Known limitations
- Mismatch detection depends on the FAA PDF header line "NOTAMs within N NM around XXX" — if the FAA changes their header format, detection will silently degrade rather than false-positive
- Alert level settings and filter preferences still reset to defaults on page reload (persistent settings planned)

---

## [1.1.1] — 2026-05-24

### Fixed
- Altitude parser now handles entries where MSL is reported as `UNKNOWN` (e.g. `UNKNOWN (650FT AGL)`). Previously the parser required a numeric MSL value before the AGL parenthetical and silently dropped these entries — they now parse correctly and appear in results or the Out of Range section as appropriate. MSL displays as `—` in the table when not reported.
- Radius mismatch detection tolerance tightened from ±5 NM to ±1 NM. The ±5 window was based on a mistaken assumption that the FAA search radius uses statute miles requiring conversion. The FAA PDF header reports the radius in NM directly, so a tight tolerance is correct.
- Help modal and CONFIGURATION.md corrected to remove inaccurate statute-miles note. The FAA search input uses miles but the PDF header states radius in NM.
