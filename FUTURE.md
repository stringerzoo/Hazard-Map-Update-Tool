# Future Development

Planned features, known limitations, and open questions. Items are loosely prioritized.

---

## Short term

### Configurable reference airport (high priority)
The reference point (lat/lon and airport ID) is hardcoded to Lebanon Muni (6I2). Making this a user-editable field — or a simple config object at the top of the file — is the primary change needed before this tool is usable by other bases.

Likely implementation: a small settings section (alongside the existing filter settings panel) with fields for Airport ID, Latitude, and Longitude. Values would be saved to `localStorage` so they persist across sessions.

**Impact:** Touches `REF` constant, the page title/header strings, and the KML `<name>` and `<description>` fields. No changes to parsing or diff logic.

### Persistent settings
Currently all threshold and filter settings reset to defaults on page reload. Saving to `localStorage` would be straightforward and would improve the monthly workflow significantly.

### Settings export/import
A way to export the current configuration as a small JSON file and re-import it — useful when sharing a configured version with another base, or after clearing browser storage.

---

## Medium term

### ForeFlight-specific KML tuning
ForeFlight has some quirks with KML rendering (icon scale, label visibility, altitude display). Testing with actual ForeFlight imports and tuning the KML output would improve the overlay quality. Specific areas to investigate:
- Whether `altitudeMode: absolute` renders correctly in ForeFlight's 2D map view
- Label visibility at different zoom levels
- Whether ForeFlight supports `<BalloonStyle>` for tap callouts

### Coordinate parser robustness
The current DMS parser handles the standard FAA 6-digit format (`DDMMSS.SSN DDDMMSS.SSW`). Some older or non-standard NOTAMs use shortened formats. A more robust parser would log skipped entries to a visible warning panel rather than silently dropping them.

### Skipped entry warnings
When a NOTAM block is found with `U/S` and `TOWER` but fails coordinate or altitude parsing, it is currently silently skipped. A "parse warnings" section in the results would let the user manually cross-check any entries that were dropped.

---

## Longer term

### Multi-base mode
Allow a single instance to manage multiple bases — useful for a BPS who covers more than one location, or for a regional coordinator. Would require a base selector UI and per-base settings storage.

### Saved baseline snapshots
Currently the diff relies on the user keeping last month's PDF. An alternative is to save the parsed tower list to `localStorage` as a baseline after each run, eliminating the need to retain the previous PDF. Would need a clear "promote current to baseline" workflow.

### Automated NOTAM retrieval
The FAA's NOTAM Management Service (NMS) has a REST API (`nms.aim.faa.gov`). If public API access becomes straightforward, the PDF drop step could be replaced with a direct API call. As of mid-2026, NMS-API access requires contacting `NOTAMS@faa.gov` — not self-service. Monitor for changes.

---

## Known limitations (v1.0.0)

- Reference airport hardcoded to 6I2
- Settings not persisted between sessions
- PDF.js requires CDN access on first load
- NOTAM ID is used as the diff key — if the FAA reissues a tower under a new NOTAM number, it will appear as Remove + Add rather than Keep (correct behavior, but worth being aware of)
- Wind turbine farms are excluded; individual turbines within a farm are not plotted
