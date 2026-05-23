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
