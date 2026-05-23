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
