# Test Methodology — NOTAM Hazard Tracker

## Purpose

This tool is used in a safety-of-flight context. Errors in parsing, filtering, or diff logic could result in a tower being missed on the hazard map or in the ForeFlight overlay. The test protocol below is designed to give confidence that the tool is producing correct, consistent, and repeatable output.

This document defines:
1. What to test and why
2. How to construct and maintain test fixtures
3. The specific test cases to run
4. What constitutes a pass
5. When to re-run the full protocol

---

## Testing philosophy

Because this is a single-file browser application with no automated test runner, all tests are **manual verification** against **known-good expected outputs**. The discipline comes from:

- Keeping real (or realistic) test PDFs as fixtures in the repo
- Documenting the expected output for each fixture pair
- Running the same cases every time a functional change is made
- Signing off in the CHANGELOG when a version passes testing

Think of it as an operational checklist, not a software test suite.

---

## Test fixtures

Fixtures live in `test/fixtures/`. They are real or anonymized NOTAM PDFs that cover specific scenarios. Maintain at minimum:

| Fixture file | Description |
|---|---|
| `sample-current.pdf` | A real obstruction NOTAM export with a known mix of towers: some above 800 ft AGL, some 600–800, some 500–600, some below 500, some outside 70 NM, cranes, and power lines. |
| `sample-previous.pdf` | An earlier export that shares some towers with `sample-current.pdf` and omits others — enabling diff testing. |
| `sample-no-qualifiers.pdf` | An obstruction NOTAM export where no towers meet the minimum AGL/range criteria. Should produce an empty result set. |

The first time you run the tool against `sample-current.pdf` and are satisfied the output is correct, save that output as `test/expected/current-only.csv`. This becomes the reference for all future runs.

**Do not modify fixture PDFs.** If a fixture becomes outdated (e.g., a NOTAM expires and you want to reflect reality), create a new fixture file and document it — don't overwrite the original.

---

## Test cases

Run all cases in order. Record pass/fail for each.

---

### TC-01 — Current month only, no previous

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Min AGL 500 ft, Max range 70 NM, Level 1 ≥ 800, Level 2 ≥ 600.

**Expected behavior:**
- No diff sections (Remove / Keep not shown)
- All qualifying towers appear in the Add section
- Summary bar shows only Total and Add counts (no Remove/Keep cells)

**Pass criteria:**
1. Tower count matches the known count from the fixture (document this number when you establish the fixture)
2. No tower with AGL < 500 ft appears in results
3. No tower with range > 70 NM appears in results
4. No crane, stack, power line, or wind turbine entry appears
5. Export CSV — open in Excel and verify row count matches on-screen count
6. Compare CSV to `test/expected/current-only.csv` — all rows must match exactly (NOTAM ID, range, bearing, MSL, AGL, location ref)

---

### TC-02 — Diff with previous month

**Setup:** Load `sample-previous.pdf` as Previous, `sample-current.pdf` as Current.  
**Settings:** Same as TC-01.

**Expected behavior:**
- Three sections: Remove, Keep, Add
- Remove: towers in previous but not current
- Keep: towers in both
- Add: towers in current but not previous
- Remove + Keep + Add = total count from TC-01 (Add + Keep = current total)

**Pass criteria:**
1. Remove + Keep count equals the qualifying tower count from the previous fixture (establish and document this)
2. Keep + Add count equals the qualifying tower count from the current fixture (TC-01 result)
3. No tower appears in more than one section
4. Export CSV — verify Action column contains only REMOVE, KEEP, or ADD
5. Compare CSV to `test/expected/diff-normal.csv`

---

### TC-03 — No qualifying towers

**Setup:** Load `sample-no-qualifiers.pdf` as Current. Leave Previous empty.  
**Settings:** Same as TC-01.

**Expected behavior:**
- Tool completes without error
- Summary shows 0 for all counts
- Add section shows "None"
- KML export produces a valid file with zero placemarks (or gracefully declines)

**Pass criteria:**
1. No JavaScript error in browser console
2. Status line reads correctly (0 qualifying towers)
3. KML file opens in a KML viewer without error

---

### TC-04 — Threshold sensitivity

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Raise Min AGL to 800 ft (so only Level 1 towers qualify).

**Pass criteria:**
1. Tower count is less than or equal to the TC-01 count
2. Every tower shown has AGL ≥ 800 ft
3. Lowering Min AGL back to 500 ft and re-running restores the TC-01 count exactly

---

### TC-05 — Alert level coloring

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Default thresholds (Level 1 ≥ 800, Level 2 ≥ 600).

**Pass criteria:**
1. In the results table, every row with AGL ≥ 800 has a red color swatch
2. Every row with AGL 600–799 has an amber swatch
3. Every row with AGL 500–599 has a blue swatch
4. Change Level 1 color to a custom color — verify the swatch in the settings panel updates immediately and the table updates after re-run

---

### TC-06 — KML structure validation

**Setup:** Complete TC-01. Export KML.

**Pass criteria:**
1. KML file opens in Google Earth (desktop or web) without error
2. Placemark count equals the qualifying tower count from TC-01
3. Tap a placemark — popup shows NOTAM ID, MSL, AGL, range, bearing, location ref
4. Icon color matches the alert level color for that tower's AGL
5. Import into ForeFlight — layer appears on map, placemarks visible, tap callout shows correct data
6. Re-run TC-01 and re-export KML — the new KML is identical to the previous (deterministic output)

---

### TC-07 — Repeatability

**Setup:** Run TC-01 twice in a row without changing any input or settings.

**Pass criteria:**
1. Tower count is identical both times
2. CSV exports are byte-for-byte identical (except the "Generated" timestamp in the last column)
3. KML exports are structurally identical (same placemarks, same coordinates)

---

### TC-08 — Edge cases (range boundary)

**Setup:** If the fixture contains a tower at or near 70 NM (within ±1 NM of the boundary), confirm it is included or excluded correctly based on its calculated range.

**Pass criteria:**
1. A tower calculated at exactly 70.0 NM is included (≤ 70 NM criterion)
2. A tower calculated at 70.1 NM is excluded
3. Changing Max range to 65 NM excludes towers that were included at 70 NM

---

## Bearing and range spot-check

For at least two towers per test run, manually verify the calculated range and bearing against an independent source (ForeFlight ruler tool, SkyVector, or manual haversine calculation from known coordinates).

The `sample-current.pdf` fixture includes `LOU 05/313 6I2` — this NOTAM references 6I2 directly (6.1NM NNW 6I2) and provides a useful sanity check: the tool should return approximately 6.1 NM at a northwesterly bearing.

Document verified spot-check results in the test log.

---

## When to run the full protocol

Run **all 8 test cases** when:
- Any change is made to the `parseTowers()` function
- Any change is made to the `computeDiff()` function
- Any change is made to `bearingRange()` or `parseDMS()`
- Any change is made to the KML export logic
- The PDF.js library version is updated
- The tool is being set up at a new base for the first time

Run **TC-01, TC-02, and TC-06 only** (abbreviated check) when:
- Only UI changes were made (colors, layout, labels)
- Only settings panel changes were made with no logic changes
- A new filter option is added but existing filter logic is unchanged

---

## Test log

Maintain a running log as a simple table appended to this file, or as a separate `test/test-log.md`.

| Date | Version | Cases run | Pass/Fail | Tester | Notes |
|---|---|---|---|---|---|
| 2026-05-23 | v1.0.0 | TC-01 through TC-08 | — | | Initial baseline — establish expected outputs |

---

## Establishing expected outputs (first run)

On the very first test run, you cannot compare against expected outputs because they don't exist yet. The procedure is:

1. Run TC-01 against `sample-current.pdf`
2. Carefully hand-verify the output against the raw PDF — spot-check at least 5 towers for correct NOTAM ID, AGL, MSL, range, and bearing
3. Verify that excluded entries (cranes, power lines, sub-500ft towers, out-of-range) are correctly absent
4. If satisfied, export the CSV and save it as `test/expected/current-only.csv`
5. Repeat for TC-02, saving `test/expected/diff-normal.csv`
6. These files now become the reference for all future runs
7. Document the tower counts and spot-checked values in the test log


---

## TC-09 — Cross-browser compatibility

Run this test case on each supported browser after any functional change, and whenever the tool is deployed to a new environment.

### Browsers to test

| Browser | Platform | Priority | Notes |
|---|---|---|---|
| Safari | macOS | Primary | Production environment for Base 173. Has `file://` font scaling quirk — sizes are deliberately over-declared to compensate. |
| Chrome | macOS | Secondary | Standard rendering, no `file://` quirks. Over-declared font sizes may render slightly large — acceptable for now, documented in `FUTURE.md`. |
| Chrome | Windows | Secondary | Current work machine environment. Same engine as macOS Chrome. |
| Edge | Windows | Secondary | Default browser on GMR-issued Windows machines — relevant if tool is shared with other BPS's. Same Chromium engine as Chrome. |
| Firefox | Any | Optional | Not common in operational environment. Test only if distribution broadens. |

### What to check in each browser

For each browser, run TC-01 (current month only) and TC-06 (KML validation) at minimum. Verify:

1. **Layout** — drop zones, settings panel, results table, and export row all render without overlap or clipping
2. **Font size** — text is comfortably readable without zooming; note any browser where sizes look significantly off
3. **PDF parsing** — PDF.js loads from CDN and processes the fixture file without error
4. **Drag and drop** — files can be dropped onto the drop zones (not just selected via click)
5. **CSV export** — file downloads correctly and opens in Excel/Numbers
6. **KML export** — file downloads correctly and is valid KML

### Known rendering differences

- **Safari (macOS, local file):** Font sizes appear smaller than declared due to `file://` scaling. Current stylesheet compensates with ~1.4× over-declaration. If upgrading to web hosting, see `FUTURE.md`.
- **Chrome/Edge (any platform):** Standard rendering. Font sizes will appear proportionally larger than on Safari local file. This is expected and acceptable in the current version.
- **All browsers:** PDF.js requires an internet connection on first load to fetch from CDN. Subsequent loads use the browser cache. Test in an offline environment only if offline use is a requirement.

### Browser test log

| Date | Version | Browser | Platform | TC-01 | TC-06 | Layout | Notes |
|---|---|---|---|---|---|---|---|
| 2026-05-23 | v1.0.0 | Safari | macOS | — | — | — | Establish baseline |
| 2026-05-23 | v1.0.0 | Chrome | Windows | — | — | — | Work machine |
| | | Edge | Windows | — | — | — | |
| | | Chrome | macOS | — | — | — | |


---

## TC-10 — NOT LGTD and stack detection

**Setup:** Requires a fixture PDF containing at least one stack with `NOT LGTD` status and at least one tower with `NOT LGTD` status (as opposed to `U/S`). Load as Current only.

**Pass criteria:**
1. Stacks with `NOT LGTD` and AGL ≥ min threshold appear in results
2. Towers with `NOT LGTD` and AGL ≥ min threshold appear in results
3. The **Unlit reason** column shows `NOT LGTD` (not `U/S`) for these entries
4. The **Type** column shows `STACK` for stack entries and `TOWER` for tower entries
5. Cranes and wind turbine farms with `NOT LGTD` do not appear in results

---

## TC-11 — Out of Range section

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Default (70 NM, 500 ft AGL).

**Pass criteria:**
1. Out of Range section appears if any entries met the height filter but exceeded 70 NM
2. Section is collapsed by default and expands on click
3. Entries in Out of Range do not appear in the Add section (no double-counting)
4. Range values in Out of Range section are all > 70 NM
5. AGL values in Out of Range section are all ≥ 500 ft
6. Out of Range entries are absent from KML export
7. Known out-of-range entries from the 5/24 fixture: HUF 03/884 (133.6 NM, 600 ft AGL) and MKL 05/228 (442 NM, 650 ft AGL, STACK, NOT LGTD)

---

## TC-12 — Mismatch detection

Run each sub-case independently.

**TC-12a — Same file in both zones**  
Load the same PDF into both Previous and Current.  
*Expected:* Amber banner appears with "same query date" warning.

**TC-12b — Mismatched reference airport**  
If a PDF queried around a different airport is available, load it as Current while configured for 6I2.  
*Expected:* Amber banner shows airport mismatch.

**TC-12c — Clean match**  
Load 5/21 PDF as Previous and 5/24 PDF as Current, tool configured for 6I2, max range 70 NM.  
*Expected:* No banner. Both PDFs are queried around 6I2 at 70 NM.

**TC-12d — Radius mismatch**  
Load any valid PDF as Current. Change Max range setting to 50 NM.  
*Expected:* Amber banner notes radius difference between PDF header (70 NM) and tool setting (50 NM).

---

## TC-13 — Help modal

**Pass criteria:**
1. Clicking "? How to use" opens the modal
2. Modal content is readable and complete (six workflow steps visible)
3. Modal closes on ✕ button click
4. Modal closes on click outside the modal panel
5. Modal closes on Escape key
6. Page content behind modal is not scrollable while modal is open
7. FAA NOTAM Search link in Step 1 opens in a new tab

---

## Note on mismatch detection reliability

The mismatch detection in TC-12 depends on the FAA PDF header format: "NOTAMs within N NM around XXX". If the FAA changes this format, detection will silently degrade. As part of TC-12c, verify the header was successfully parsed by confirming **no false-positive banner** appears on a known-good pair of PDFs. If the banner is absent on a clean pair, the parser is working. If it fires unexpectedly, inspect the raw PDF header text.
