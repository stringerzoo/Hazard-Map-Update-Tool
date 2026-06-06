# NOTAM Hazard Tracker

A standalone browser-based tool for AEL EMS helicopter bases to track unlit obstruction NOTAMs, maintain physical hazard maps, and generate ForeFlight moving-map overlays.

## What it does

Each month, EMS helicopter crews are required to maintain a base hazard map showing unlit towers within their operational area. This tool automates the analysis step: given one or two FAA obstruction NOTAM PDF exports, it parses the reports, identifies which towers need to be added or removed from the hazard map, and exports a KML file for direct import into ForeFlight.

**Inputs:** FAA NOTAM PDF exports (obstruction-filtered, geography search around the base)

**Outputs:**
- Remove / Keep / Add diff summary for hazard map updates
- Buffer Zone — towers below the minimum threshold but within a configurable safety margin (KML only)
- Out of Range — towers the FAA query returned beyond the stated radius (audit trail, not plotted)
- CSV export with full detail
- KML overlay for ForeFlight, color-coded by alert level

## How to use

1. Go to [FAA NOTAM Search](https://notams.aim.faa.gov/notamSearch/) and run a Geography search centered on your base airport, **70 NM radius**, filtered for **Obstruction**. Export as PDF.
2. Open `notam-hazard-tracker.html` in any modern browser (Chrome, Safari, Firefox, Edge).
3. In **Filter & Alert Settings**, confirm your base is selected. 156 AEL bases are pre-loaded — search by base number, name, city, or airport code. Use the manual override for bases not yet in the list.
4. Drop the current month's PDF into the **Current month** zone. Optionally drop last month's PDF into **Previous month** to enable the diff.
5. Click **Parse & Report**.
6. Review the Remove / Keep / Add sections and update your physical hazard map.
7. Click **Export KML for ForeFlight** and import the file in ForeFlight: Maps → Layers → Import File. Replace the previous month's layer.

The built-in **? How to use** button in the header covers the full workflow with operational notes.

## No installation required

This is a single self-contained HTML file. No server, no framework, no dependencies to install. PDF.js is loaded from CDN on first open and cached by the browser thereafter — subsequent runs work without internet access.

All settings (base selection, thresholds, alert colors) are saved to browser `localStorage` and restored on next load.

## Repository structure

```
notam-hazard-tracker/
├── notam-hazard-tracker.html   # The application (single file, self-contained)
├── README.md                   # This file
├── CHANGELOG.md                # Version history
├── CONFIGURATION.md            # Settings reference and operational rationale
├── TESTING.md                  # Test methodology and test cases (TC-01 through TC-16)
├── FUTURE.md                   # Planned features, known limitations, update workflow
├── ael_bases_review.csv        # Human-readable base coordinate reference (156 AEL bases)
└── test/
    ├── fixtures/               # NOTAM PDF fixtures for repeatable testing
    │   ├── sample-current.pdf
    │   ├── sample-previous.pdf
    │   └── sample-no-qualifiers.pdf
    └── expected/               # Expected CSV outputs for each fixture pair
        ├── current-only.csv
        ├── diff-normal.csv
        └── diff-no-qualifiers.csv
```

## Base configuration

156 AEL bases are pre-loaded with coordinates sourced from the Feb 2025 company KML file and manually verified. Select your base from the dropdown in Filter & Alert Settings — no code editing required. See `CONFIGURATION.md` for coordinate format, manual override instructions, and the process for updating base data.

## Operational thresholds

Default settings reflect Base 173 operational standards (FAR Part 135 / company GOM):

| Setting | Default | Rationale |
|---|---|---|
| Min AGL | 500 ft | Night obstacle clearance minimum |
| Buffer | 50 ft | Safety margin — towers 450–499 ft AGL appear in KML only |
| Max range | 70 NM | Matches FAA query radius |
| Level 1 | ≥ 800 ft AGL | Critical |
| Level 2 | ≥ 600 ft AGL | Caution |
| Level 3 | ≥ 500 ft AGL | Advisory |

See `CONFIGURATION.md` for the full operational rationale, including the buffer zone design and the Out of Range section explanation.

## Testing

Before relying on any version of this tool for safety-of-flight purposes, run the full test protocol defined in `TESTING.md`. Re-run after any functional change to parsing, filtering, diff, or export logic. The test suite covers 16 test cases including cross-browser compatibility (TC-09) and buffer zone behavior (TC-14).

## Multi-base use

The tool is architecturally designed for multi-base use — 156 AEL bases are pre-loaded, and each user selects their own base from the dropdown with no code changes required. However, the tool has not yet completed full test validation beyond Base 173 and should not be distributed to other bases until the test protocol in `TESTING.md` has been completed and signed off. See `FUTURE.md` for the base lookup table update workflow and longer-term distribution plans.

## License

Internal use — Air Evac Lifeteam / Global Medical Response. Not for public distribution.
