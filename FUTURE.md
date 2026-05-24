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


---

## Web hosting considerations

The app is currently designed to run as a local `file://` HTML file on macOS. If it is ever hosted on a web server (e.g., GitHub Pages, a base intranet, or a shared hosting service), the following will need to be revisited:

### Font sizing
Safari treats `file://` URLs differently from `http://` — the viewport meta tag is largely ignored for desktop layout, and Safari applies its own internal scaling heuristic. To compensate, font sizes in the current version are deliberately over-declared (scaled ~1.4× from design values) so they render correctly as a local file.

When hosted over HTTP, those inflated sizes will render too large. The fix is to reset all font sizes to their intended design values (roughly divide current px values by 1.4) and rely on the standard viewport meta tag, which will be respected normally in a served context.

**Impact:** CSS only — no logic changes required. Should be a single find-and-replace pass on the stylesheet.

### PDF.js CDN dependency
The app loads PDF.js from `cdnjs.cloudflare.com`. This works fine for both local and hosted use as long as internet access is available, but a hosted version intended for offline or intranet use should bundle PDF.js locally.

### CORS / file access
No server-side code is needed — the app is entirely client-side. Any static hosting (GitHub Pages, S3, Nginx) will work without modification beyond the font size adjustment above.


---

## Persistent settings

Alert level thresholds and filter values reset to defaults on every page reload. This is the most frequently encountered friction point in the monthly workflow. Saving to `localStorage` is straightforward and would mean settings survive between sessions without any server-side infrastructure.

Items to persist: min AGL, max range, Level 1/2/3 AGL thresholds, Level 1/2/3 colors.

A "reset to defaults" button should accompany any persistent settings implementation.


---

## Base lookup table management

The base lookup table is currently embedded directly in the HTML file as a JavaScript array (`BASE_LIST`). This means adding or updating a base requires editing the HTML — workable for occasional changes but not scalable as the fleet grows or bases move.

### Near-term: lookup table editor

A built-in editor panel would allow authorized users to:
- Add a new base (ID, name, city, state, airport code, coordinates)
- Edit an existing base's coordinates or airport code
- Mark a base as inactive (hide from dropdown without deleting)
- Export the current table as JSON for backup or sharing

Changes would be saved to `localStorage` as an override layer on top of the embedded table, so the base HTML file doesn't need to be modified. A "restore embedded defaults" option would clear any local overrides.

### Medium-term: external lookup file

Move the base table to a separate `ael_bases.json` file loaded at runtime. This separates base data from application code — updating coordinates requires only replacing the JSON file, not touching the HTML. Requires either web hosting (so the JSON can be fetched via HTTP) or a packaging step that re-embeds the JSON into the HTML.

### Longer-term: shared/synced table

If the tool is deployed to multiple bases, each base independently maintaining its own lookup table creates drift. A shared source of truth — a hosted JSON file, a simple API, or a shared document — would ensure all bases see the same data. Access control and an audit trail would be needed given the safety-of-flight context.

### Current state (v1.4.0)

The base table is embedded directly in the HTML as the `BASE_LIST` JavaScript array. **The HTML file is the source of truth.** The `ael_bases_lookup.json` file was an intermediate artifact from the initial KML extraction workflow and is no longer needed — it can be deleted or archived.

The `ael_bases_review.csv` is a human-readable reference only — useful for spot-checking coordinates or sharing with others, but it does not feed back into the tool.

### Interim update workflow (current)

Until a built-in editor is available, base table updates follow this process:

1. Edit `ael_bases_review.csv` with corrected or new coordinates (add rows, fix existing entries, add airport codes)
2. Send the updated CSV to Claude in a new conversation
3. Claude re-generates the embedded `BASE_LIST` in the HTML and produces an updated `notam-hazard-tracker.html`
4. Replace the HTML in the repo and commit

The CSV is therefore the **human-editable source** for base data, and the HTML is the **deployed artifact**. Keep the CSV in the repo so it stays in sync with the embedded table.

New base entries in the CSV should follow the existing column structure:

| Column | Example | Notes |
|---|---|---|
| Base ID | `AE 189` | Normalized format — `AE NNN` |
| Base Name | `Anadarko` | Short name as used operationally |
| City | `Anadarko` | Physical city location |
| State | `OK` | Two-letter abbreviation |
| Airport | `KADHK` | FAA identifier; leave blank if unknown |
| Lat (DD) | `35.052700` | Decimal degrees |
| Lon (DD) | `98.094400` | Decimal degrees, positive (W implied) |
| Lat (DD MM.mmmm) | `35 03.1620` | For reference; Claude recalculates from DD |
| Lon (DD MM.mmmm) | `98 05.6640` | For reference; Claude recalculates from DD |
| Source | `Verified (KADHK)` | How coordinates were obtained |
| Notes | `` | Any flags or caveats |
