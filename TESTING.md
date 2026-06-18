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
| `notams-obst-6ky1-2026-06-05.pdf` | Real 6KY1 report — used for reissue detection testing (TC-18). |
| `notams-obst-6ky1-2026-06-18.pdf` | Real 6KY1 report — used for reissue detection testing (TC-18). Contains known reissue, genuine remove, and genuine add relative to the June 5 report. |

The first time you run the tool against `sample-current.pdf` and are satisfied the output is correct, save that output as `test/expected/current-only.csv`. This becomes the reference for all future runs.

**Do not modify fixture PDFs.** If a fixture becomes outdated, create a new fixture file and document it — don't overwrite the original.

---

## Test case categories

Test cases fall into three categories:

### 🔁 Regression tests
Run these before every release and after any functional change to parsing, filtering, diff, reissue detection, or export logic. These are the core safety checks.

> **TC-01, TC-02, TC-03, TC-04, TC-05, TC-07, TC-08, TC-09, TC-10, TC-11, TC-14, TC-14b, TC-18, TC-18b**

### ⚙️ Conditional tests
Run these only when the relevant subsystem has changed. Safe to skip if the affected area was not touched.

| Test | Run when... |
|---|---|
| TC-06 | KML export logic or ForeFlight import workflow changes |
| TC-12 | Mismatch detection or PDF header parsing changes |
| TC-13 | Modal content or modal interaction behavior changes |
| TC-15, TC-15b | Base selector, settings persistence, or coordinate handling changes |

### 📋 Historical record
These tests were one-time validations for specific releases. They document what was verified at the time and do not need to be re-run. Retained for audit trail purposes.

> **TC-16** (v1.4.0 UI), **TC-17** (v1.4.1 regression), **TC-18c** (v1.6.0 arrow placement)

---

## Pre-release checklist

Before tagging any release for operational use:

- [ ] All 🔁 Regression tests pass
- [ ] Relevant ⚙️ Conditional tests pass for this release's changes
- [ ] No JavaScript console errors during any test run
- [ ] CHANGELOG updated with version, date, and test sign-off
- [ ] Version number matches in HTML comment, header display, and Git tag

---

## Test cases

---

### TC-01 — Current month only, no previous 🔁

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Min AGL 500 ft, Max range 70 NM, Level 1 ≥ 800, Level 2 ≥ 600.

**Expected behavior:**
- No diff sections (Remove / Keep not shown)
- All qualifying towers appear in the Add section
- Summary bar shows only Total and Add counts

**Pass criteria:**
1. Tower count matches the known count from the fixture
2. No tower with AGL < 500 ft appears in results
3. No tower with range > 70 NM appears in results
4. No crane, stack, power line, or wind turbine entry appears
5. Export CSV — row count matches on-screen count
6. Compare CSV to `test/expected/current-only.csv` — all rows match exactly

---

### TC-02 — Diff with previous month 🔁

**Setup:** Load `sample-previous.pdf` as Previous, `sample-current.pdf` as Current.  
**Settings:** Same as TC-01.

**Pass criteria:**
1. Remove + Keep count equals the qualifying tower count from the previous fixture
2. Keep + Add count equals the qualifying tower count from the current fixture
3. No tower appears in more than one section
4. Export CSV — Action column contains only REMOVE, KEEP, ADD, REISSUED, or BUFFER
5. Compare CSV to `test/expected/diff-normal.csv`

---

### TC-03 — No qualifying towers 🔁

**Setup:** Load `sample-no-qualifiers.pdf` as Current. Leave Previous empty.  
**Settings:** Same as TC-01.

**Pass criteria:**
1. No JavaScript error in browser console
2. Status line reads correctly (0 qualifying towers)
3. KML file opens in a KML viewer without error

---

### TC-04 — Threshold sensitivity 🔁

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Raise Min AGL to 800 ft.

**Pass criteria:**
1. Tower count is less than or equal to the TC-01 count
2. Every tower shown has AGL ≥ 800 ft
3. Lowering Min AGL back to 500 ft restores the TC-01 count exactly

---

### TC-05 — Alert level coloring 🔁

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Default thresholds.

**Pass criteria:**
1. Every row with AGL ≥ 800 has a red swatch
2. Every row with AGL 600–799 has an amber swatch
3. Every row with AGL 500–599 has a blue swatch
4. Changing Level 1 color updates the swatch immediately and the table after re-run

---

### TC-06 — KML structure validation ⚙️

**Setup:** Complete TC-01. Export KML.

**Pass criteria:**
1. KML file opens in Google Earth without error
2. Placemark count equals the qualifying tower count from TC-01
3. Tap a placemark — popup shows NOTAM ID, MSL, AGL, range, bearing, location ref
4. Icon color matches the alert level color for that tower's AGL
5. Import into ForeFlight — layer appears on map, tap callout shows correct data
6. Re-export KML — output is structurally identical (deterministic)

---

### TC-07 — Repeatability 🔁

**Setup:** Run TC-01 twice in a row without changing any input or settings.

**Pass criteria:**
1. Tower count is identical both times
2. CSV exports are byte-for-byte identical (except the "Generated" timestamp)
3. KML exports are structurally identical

---

### TC-08 — Edge cases (range boundary) 🔁

**Setup:** Fixture must contain a tower at or near 70 NM.

**Pass criteria:**
1. A tower calculated at exactly 70.0 NM is included
2. A tower calculated at 70.1 NM is excluded
3. Changing Max range to 65 NM excludes towers included at 70 NM

---

## Bearing and range spot-check

After any change to the coordinate parser or haversine calculation, manually verify range and bearing for at least two towers using an independent tool (ForeFlight, SkyVector, or a navigation calculator). Choose one near tower and one far tower.

Expected precision: range within ±0.1 NM, bearing within ±1°.

---

### TC-09 — Cross-browser compatibility 🔁

**Setup:** Run TC-01 in each supported browser.

**Browsers:** Chrome (primary), Safari (macOS local file), Firefox, Edge.

**Pass criteria:**
1. Results are identical across browsers
2. No console errors in any browser
3. PDF drop works in Safari (page does not navigate away)
4. File picker works in all browsers

---

### TC-10 — Coordinate parser edge cases 🔁

Test against real NOTAM entries with non-standard formats.

**Pass criteria:**
1. `UNKNOWN (650FT AGL)` format parsed correctly (MSL shown as —, AGL correct) — verified against MKL 05/228
2. Altitude with decimal feet (e.g. `1345.8FT (319.9FT AGL)`) parsed correctly
3. Coordinates with decimal seconds (e.g. `351839.00N0931406.00W`) parsed correctly
4. STACK type entries included; CRANE, WIND TURBINE, POWER LINE entries excluded
5. NOT LGTD trigger includes entries that lack U/S
6. U/S trigger includes entries that lack NOT LGTD

---

### TC-11 — Out of Range audit trail 🔁

**Setup:** Load `notams-obst-6ky1-2026-06-05.pdf` as Current. Base set to AE 173 (6KY1), max range 70 NM.

**Pass criteria:**
1. Out of Range section is present and collapsed by default
2. Expanding it shows entries with AGL ≥ buffer floor but range > 70 NM
3. Out of Range entries do not appear in the main results or KML
4. Out of Range entries do not appear in the CSV main result rows
5. Section header tooltip explains why the section exists
6. Known out-of-range entries from the 5/24 fixture: HUF 03/884 (133.6 NM, 600 ft AGL) and MKL 05/228 (442 NM, 650 ft AGL, STACK, NOT LGTD)

---

### TC-12 — Mismatch detection ⚙️

**TC-12a — Same file in both zones**  
Load the same PDF into both Previous and Current.  
*Expected:* Amber banner with "same query date" warning.

**TC-12b — Mismatched reference airport**  
Load a PDF queried around a different airport as Current while tool is configured for 6KY1.  
*Expected:* Amber banner shows airport mismatch.

**TC-12c — Clean match**  
Load June 5 PDF as Previous, June 18 PDF as Current. Tool configured for 6KY1, max range 70 NM.  
*Expected:* No banner.

**TC-12d — Radius mismatch**  
Load any valid PDF as Current. Change Max range to 50 NM.  
*Expected:* Amber banner notes radius difference.

**Note:** If TC-12c unexpectedly shows a banner, the PDF header parser may have broken. Inspect the raw PDF header text.

---

### TC-13 — Help modal ⚙️

**Pass criteria:**
1. "? How to use" opens the modal
2. All six workflow steps visible and readable
3. Reissued section described in Step 4
4. Modal closes on ✕, outside click, and Escape key
5. FAA NOTAM Search link opens in a new tab
6. GitHub repo link at bottom opens in a new tab

---

### TC-14 — Buffer zone 🔁

**Setup:** Load `sample-current.pdf` as Current. Leave Previous empty.  
**Settings:** Min AGL 500 ft, Buffer 50 ft. Confirm buffer floor = 450 ft.

**Pass criteria:**
1. Towers with AGL 450–499 ft appear in Buffer Zone, not in Add
2. Towers with AGL ≥ 500 ft appear in Add as normal
3. Towers with AGL < 450 ft do not appear anywhere
4. Buffer Zone section is collapsed by default
5. Summary bar shows Buffer count when buffer entries present
6. KML includes buffer entries with buffer color (default gray)
7. CSV includes buffer entries with action `BUFFER`
8. Setting buffer to 0 ft: 450–499 ft towers disappear entirely
9. Setting buffer to 200 ft: captures towers down to 300 ft AGL

**TC-14b — Buffer does not affect diff logic** 🔁  
Load June 5 as Previous, June 18 as Current.  
Confirm buffer entries appear only in Buffer Zone, not in Remove/Keep/Add.

---

### TC-15 — Base selector ⚙️

**Pass criteria:**
1. Default base loads as AE 173, header shows `NOTAM Obstruction Hazard Tracker // AE 173 · Lebanon, KY`
2. Search by base number, name, city, and airport code all work
3. Selecting a base updates coordinates and header
4. Coordinates display in DD MM.dd format
5. Saving and reloading restores selected base

**TC-15b — Manual override** ⚙️  
1. Enter known coordinates in manual override fields
2. Click Apply override — header updates to DD MM N DDD MM W format
3. Save and reload — manual override state restored

---

### TC-18 — Reissue detection 🔁

**Setup:** Load `notams-obst-6ky1-2026-06-05.pdf` as Previous, `notams-obst-6ky1-2026-06-18.pdf` as Current. Base AE 173, default thresholds.

**Pass criteria:**
1. Summary bar: Total 7 · Remove 1 · Keep 5 · Reissued 1 · Add 1 · Buffer 1
2. Summary bar order: Total, Remove, Keep, Reissued, Add, Buffer
3. Reissued section appears between Keep and Add
4. Reissued section shows LEX 04/065 → LEX 06/042, 47.4 NM, 530 ft AGL
5. Remove section shows HUF 06/101 (961 ft AGL, 55.0 NM)
6. Add section shows LEX 06/018 (533 ft AGL, 47.6 NM)
7. Out of Range shows 3 entries (MKL 05/228, HUF 03/884, HUF 04/651)
8. CSV includes `REISSUED,LEX 04/065 -> LEX 06/042,...`

**TC-18b — No false reissues on single-file run** 🔁  
Load only the June 18 PDF. Confirm no Reissued section and no Reissued cell in summary bar.

---

## Historical test records

The following tests were one-time validations for specific releases. They do not need to be re-run unless the relevant feature is revisited.

### TC-16 — v1.4.0 UI validation 📋
*(One-time, 2026-05-24)*
- Button reads Parse & Report
- Mismatch banner appears below Parse & Report button
- Safari drop zone behavior correct
- Dropping outside drop zones does nothing

### TC-17 — v1.4.1 regression check 📋
*(One-time, 2026-05-25)*
- Header format: `NOTAM Obstruction Hazard Tracker // AE 173 · Lebanon, KY` with version below
- Coordinates display DD MM.dd (2 decimal places)
- Column header tooltips present on Range, Bearing, Location ref
- Color pickers vertically aligned in Settings
- Modal Step 1: no disclaimer step, mentions hospital identifier and Lat/Lon
- Modal Step 2: mentions "differences analysis"
- Modal Step 5: CSV purpose explained, print-to-PDF tip present

### TC-18c — v1.6.0 arrow placement 📋
*(One-time, 2026-06-18)*  
All result section headers show ▶ toggle arrow on the LEFT, before the section title.

---

## Testing notes log

### 2026-05-25 (v1.4.1)

| # | Issue | Resolution |
|---|---|---|
| 1 | Apply Override purpose unclear | Modal Step 3 clarified — fields affect display only, not base list |
| 2 | Range/Bearing columns lack context | Hover tooltips added |
| 3 | Header layout unclear | Redesigned: app name // base · location, version below |
| 4 | Step 1 disclaimer unnecessary; identifier too narrow | Removed; updated to include hospital and Lat/Lon |
| 5 | "hazard map diff" unclear | Reworded to plain language |
| 6 | Step 2 didn't describe diff output | Updated to mention add/keep/remove |
| 7 | KML to ForeFlight transfer complicated on company iPads | Documented in FUTURE.md |
| 8 | 4dp coordinate precision unnecessary | Reduced to 2dp (DD MM.dd) |
| 9 | Color pickers misaligned | Fixed with min-width on column |
| 10 | CSV purpose not explained | Explanation added to modal Step 5 |

### 2026-06-18 (v1.6.0)

| # | Issue | Resolution |
|---|---|---|
| 1 | Drop zones non-functional after 1.6 update | Orphan `sCell(cu` fragment from partial replacement caused JS syntax error |
| 2 | Remove section missing from results | `makeSection(d.remove,...)` call dropped during replacement pass |
| 3 | Reissued label rendering after hint text | Element order corrected: toggle → title → count → hint |
| 4 | Reissued summary cell after Add | Cell order corrected to match section render order |
| 5 | Toggle arrows right-aligned on result sections | Moved to left of all result section headers |
