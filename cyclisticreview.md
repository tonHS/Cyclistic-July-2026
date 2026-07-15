# Cyclistic Capstone 2026 — notebook review

21 cells · 12 months of Divvy trip data · 2026-07-15

**Bottom line.** The pipeline is sound and the code runs. Two issues change what the analysis is allowed to conclude: the 26-hour outliers are a Divvy auto-termination artifact inflating the duration means, and every "casual rider vs member rider" claim is really about *rides*, not people — the dataset has no user identifier.

---

## Analytical errors & interpretation

### 1. "Rides" vs "riders" conflation
*cell 8, and narrative throughout*

The dataset has one row per ride with a `ride_id` but no rider identifier. Statements like "0.36 of all unique rides were casual riders" and "casual riders use bikes for non-work outings" describe rides, not people. That matters because the Cyclistic case is about converting casual *users* to members — one very active casual user contributes many rides and can distort every behavioural pattern. Every takeaway should be phrased at the ride level, with a note that person-level conversion cannot be measured from this data.

### 2. Max-duration outliers are a Divvy system artifact, not real rides
*cells 15, 16*

Casual max = `1 day 01:59:57`, member max = `1 day 01:59:54`. Both are ~25:59:59 — the known Divvy auto-termination / lost-bike cap. Cell 16 confirms "1 ride each" at the max but doesn't check the shape of the upper tail. Many rides are almost certainly bunched near 24–26h and inflating the mean. Either exclude rides above ~24h or make the medians the headline (11.16 casual vs 8.59 member).

### 3. Zero and near-zero durations dropped silently
*cell 15*

The filter `> 0` removes rows without reporting how many, and the printed min is still `0.0` (a sub-second ride rounded down). Divvy also has known "test unlock" rides that look legitimate but are a few seconds long — another artifact worth quantifying.

### 4. Sorting bug on the monthly and daily duration tables
*cells 17, 18*

Both tables print alphabetically (April, August, December… / Friday, Monday, Saturday…) instead of chronologically or Mon–Sun. Cell 12 solves this for daily counts using an ordered categorical, but the same fix is missing here — so the tables meant to reveal the seasonal and weekend duration patterns are unreadable in the current output.

### 5. Column drop discards all distance analysis
*cell 6*

Cell 6 drops `start_lat`, `start_lng`, `end_lat`, `end_lng`. Distance — and round-trip vs point-to-point — is one of the strongest ways to distinguish leisure casual rides from commuter member rides, and it's essentially the question the case asks. `end_station_name` is retained but never analyzed.

### 6. Missing end_station_name not quantified
*cells 4, 20*

The tail sample in cell 4 shows 4 of 5 rows with a NaN `end_station_name`. That pattern — electric bikes ending off-station — is systematic in Divvy and can hit double-digit percentages of rides. The station-level analysis in cell 20 silently excludes them, and the magnitude is never reported.

### 7. Round-trip behaviour not examined
*cell 20*

The single largest known behavioural marker in this dataset is that casual riders take lakefront loops (Navy Pier → Navy Pier) much more often than members. The analysis stops at "top start stations" and never joins start–end pairs. The top casual stations shown (Navy Pier, DuSable Lake Shore, Millennium Park, Shedd Aquarium, Theater on the Lake) are all tourist landmarks — substantive gap not to profile them.

### 8. Third bike type may be missing
*cell 19*

Cell 19 shows only `classic_bike` and `electric_bike`. Divvy has historically included `docked_bike` (and briefly `electric_scooter`). If those were never in this 12-month slice, fine — but the notebook doesn't verify it, and if they were, they were silently dropped somewhere.

### 9. Monthly-counts commentary isn't grounded in a table
*cell 11*

The annotation asserts "over 400,000 member rides each month from May–October" and "casual rides exceeded 300,000 in June, July, August," but no numeric monthly table is printed — only the chart. Given member total ≈ 3.82M / 12 ≈ 318K average, "over 400K for 6 months" is a strong claim to eyeball from a bar chart. Print the table.

---

## Data handling & robustness

### 10. `glob.glob(...)[:12]` is non-deterministic
*cell 2*

Glob returns filesystem order. The head/tail checks in cells 3–4 happen to show July 2025 → June 2026, but a 13th file added to the folder would silently swap the subset. Sort the file list explicitly and/or assert on the resulting date range.

### 11. Timezone / DST not handled
*cells 10, 13*

`started_at` is parsed as naive datetime. The "hour of day" and morning-commute analyses conflate CDT and CST rides. Small effect — flag it if you make claims about the exact commute peak.

### 12. Redundant work and no measure of variability
*cells 8, 15, 17, 18*

Cell 8 rebuilds `df_casual` / `df_member` that cell 7 already built. After dedup, `nunique() == shape[0]` by construction, so the repeated `.nunique()` prints in cell 8 add nothing. `.head(12)` in cells 17–18 is a no-op — the Series has exactly 12 (or 7) rows.

Every duration comparison is reported as a point estimate with no measure of spread. A standard deviation, IQR, or even a small histogram would materially strengthen the "casual rides are longer" story.

---

## Small nits

### 13. Comment copy
*cells 8, 12*

Cell 12 comment says "MOnday-Friday." Cell 8 wording says "casual riders" repeatedly when it means "casual rides" — clean that up wherever the same conflation appears in the narrative comments (see finding 1).

---

## Fix these two before drawing conclusions

1. **Cap or investigate the ~26-hour outliers** and refresh the duration means — or promote the median to be the headline.
2. **Fix the alphabetical sort in cells 17–18** so the monthly and weekly duration tables are actually readable. The strongest seasonal / weekday-vs-weekend claim in the notebook is currently being made from a chart rather than the corresponding numeric tables.

Everything else — round trips, distance from lat/lng, end-station NaN rate, `docked_bike` — is opportunity rather than error. But the ride-vs-rider caveat should sit somewhere prominent, because it changes what the whole analysis is allowed to conclude.
