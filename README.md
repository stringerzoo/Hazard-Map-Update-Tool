# NOTAM Hazard Tracker

A standalone browser-based tool for EMS helicopter bases to track unlit obstruction NOTAMs and maintain hazard maps and ForeFlight overlays.

## What it does

Each month, EMS helicopter crews are required to maintain a base hazard map showing unlit towers within their operational area. This tool automates the analysis step: given one or two FAA obstruction NOTAM PDF exports, it identifies which towers need to be added or removed from the map, and exports a KML file for direct import into ForeFlight as a moving-map overlay.

**Inputs:** FAA NOTAM PDF exports (obstruction-filtered, geography search around the base)  
**Outputs:** Remove / Keep / Add diff summary · CSV export · KML overlay for ForeFlight

## How to use

1. Go to [FAA NOTAM Search](https://notams.aim.faa.gov/notamSearch/) and run a Geography search centered on your airport, 70 NM radius, filtered for Obstruction. Export as PDF.
2. Open `notam-hazard-tracker.html` in any modern browser (Chrome, Safari, Firefox).
3. Drop the current month's PDF into the **Current month** zone. Optionally drop last month's PDF into **Previous month** for a diff.
4. Adjust filter and alert level settings if needed.
5. Click **Run diff**.
6. Review the Remove / Keep / Add sections and update your physical hazard map accordingly.
7. Export KML and import into ForeFlight (Maps → Layers → Import File).

## No installation required

This is a single self-contained HTML file. No server, no dependencies to install, no internet connection required at runtime (PDF.js is loaded from CDN on first open; after that, browser caching handles it).

## Repository structure

```
notam-hazard-tracker/
├── notam-hazard-tracker.html   # The application (single file)
├── README.md                   # This file
├── CHANGELOG.md                # Version history
├── CONFIGURATION.md            # Base-specific settings reference
├── TESTING.md                  # Test methodology and test cases
├── FUTURE.md                   # Planned features and known limitations
└── test/
    ├── fixtures/               # Sample NOTAM PDFs for repeatable testing
    │   ├── sample-current.pdf
    │   ├── sample-previous.pdf
    │   └── sample-no-qualifiers.pdf
    └── expected/               # Expected outputs for each fixture pair
        ├── current-only.csv
        ├── diff-normal.csv
        └── diff-no-qualifiers.csv
```

## Configuration

Default settings are tuned for AEL Base 173 (Lebanon Muni, 6I2). See `CONFIGURATION.md` for all user-adjustable parameters and the short-term roadmap for making the tool base-agnostic.

## Testing

Before relying on any version of this tool for safety-of-flight purposes, run the test protocol defined in `TESTING.md`. This should be repeated after any functional change to the parsing, filtering, diff, or export logic.

## License

Internal use — AEL Base 173. See `FUTURE.md` for multi-base distribution plans.
