# `16_transition_analysis.R`

## Inputs
- `outputs/<VERSION>/transition/TM_<YEARFROM><YEARTO>_<GEOID>.csv`: baseline county × year-pair transition matrices, from `02_TM_disag.R`.
- `outputs/<VERSION>/transition_final/TM_<YEARFROM><YEARTO>_<GEOID>.csv`: corrected county × year-pair transition matrices, from `15_Transition_final.R`.

## Outputs
- `figures/<VERSION>/transition_comparison/p_3x3_rownorm_allcounties.png`: all-counties-pooled row-normalized 3×3 transition matrix, Baseline vs. Corrected, across 4 periods. Main transitions figure.
- `figures/<VERSION>/transition_comparison/p_3x3_rownorm_plaintiffpooled.png`: same figure, pooled across just the (subset of) plaintiff counties. There is not all plaintiff counties yet, but one can increase the list by adding the GEOIDs in the `PLAINTIFF_GEOIDS` list.
- `figures/<VERSION>/transition_comparison/plaintiff_counties/<GEOID>/p_3x3_rownorm_<GEOID>.png`: same figure, per individual plaintiff county.

## Summary
This is the script that produces the transition matrices. It compares how often GM/Tolerant/Vulnerable pixels switch category year-to-year, before vs. after the post-processing correction, restricted specifically to the three Dicamba-relevant categories (NonCrop, Unclassified, and NoData are dropped from the analysis entirely). The same 3×3 row-normalized matrices is produced three times at different levels of pooling: one per plaintiff county, pooled across all plaintiff counties, and pooled across every county nationally. On each figures, there is a baseline and corrected version of the matrix pooled across all year, but also for the 2015-2016 years, 2016-2017 years, and the 2017-2018 years, allowing to see the change accross years and also the effect of the correction on the transitions.

## Detailed explanation

### Loading every transition matrix into one long table

```r
read_all_tms <- function(base_dir, source_label) {
  files <- list.files(base_dir, pattern = "^TM_.*\\.csv$", recursive = TRUE, full.names = TRUE)
  ...
  purrr::map_dfr(files, function(f) {
    fname <- basename(f)
    parts <- str_match(fname, "^TM_(\\d{4})(\\d{4})_(\\d+)\\.csv$")
    if (any(is.na(parts))) return(NULL)
    
    year_from <- as.integer(parts[2])
    year_to   <- as.integer(parts[3])
    geoid     <- parts[4]
    
    tm <- tryCatch(as.matrix(read.csv(f, row.names = 1, check.names = FALSE)), error = function(e) NULL)
    if (is.null(tm)) return(NULL)
    
    as.data.frame(tm) %>%
      rownames_to_column("from_category") %>%
      pivot_longer(-from_category, names_to = "to_category", values_to = "n_pixels") %>%
      mutate(GEOID = geoid, year_from = year_from, year_to = year_to, source = source_label)
  })
}
```
The year-from/year-to/GEOID are pulled directly out of the filename itself via `str_match()`. For instance, `TM_20132014_08027.csv` becomes `year_from = 2013`, `year_to = 2014`, `geoid = "08027"`. Each matrix is read as a wide `data.frame`, given back its row names as an explicit `from_category` column (`rownames_to_column`), then pivoted from wide (one column per `to_category`) to long (`from_category, to_category, n_pixels`), allowing us to stack matrices from every county/year-pair/source into one unified long table with `map_dfr()`.

### Restricting to counties present in both sources

```r
baseline_counties <- unique(tm_baseline_long_raw$GEOID)
final_counties     <- unique(tm_final_long_raw$GEOID)
common_counties    <- intersect(baseline_counties, final_counties)
...
tm_baseline_long_raw <- tm_baseline_long_raw %>% filter(GEOID %in% common_counties)
tm_final_long_raw    <- tm_final_long_raw    %>% filter(GEOID %in% common_counties)
```
Since the baseline transition matrices (`02_TM_disag.R`) are the full set of counties while the corrected ones (`15_Transition_final.R`) were produced only for a county with a model assignment, the two sets of available counties aren't guaranteed to match exactly. Therefore, we use only common counties.
This restriction is only made so Baseline-vs-Corrected comparison is possible. Neverless, the full baseline transition matrices (transition/TM_*.csv, from 02_TM_disag.R) remain available for every county with no other best model than baseline.

### Restricting to the 3×3 GM/Tolerant/Vulnerable block

```r
tm_3x3_long <- bind_rows(tm_baseline_long_raw, tm_final_long_raw) %>%
  filter(from_category %in% CROP_CATEGORIES, to_category %in% CROP_CATEGORIES) %>%
  mutate(from_category = factor(from_category, levels = CROP_CATEGORIES),
         to_category   = factor(to_category, levels = CROP_CATEGORIES))
```
Both filters (`from_category` and `to_category` must both be in `CROP_CATEGORIES = c("GM", "Tolerant", "Vulnerable")`) are applied before any row-share percentage is computed anywhere downstream. Hence, every row-share percentage computed in this script is explicitly "share of pixels that stayed within the 3 Dicamba-relevant categories". `factor(..., levels = CROP_CATEGORIES)` sets a consistent GM/Tolerant/Vulnerable ordering for every plot built from `tm_3x3_long`.

### Three levels of pooling

The same computation: group by category pair (and possibly county), sum pixels, then compute each row's percentage share (`row_share_pct = 100 * n_pixels / sum(n_pixels)` within each `from_category` group), is applied three times at different granularities:

```r
compute_period_row_shares <- function(period_years, geoid_filter) {
  df <- tm_3x3_plaintiff %>% filter(GEOID %in% geoid_filter)
  if (!is.null(period_years)) {
    df <- df %>% filter(year_from == period_years[1], year_to == period_years[2])
  }
  if (nrow(df) == 0) return(tibble())
  df %>%
    group_by(GEOID, source, from_category, to_category) %>%
    summarise(n_pixels = sum(n_pixels, na.rm = TRUE), .groups = "drop") %>%
    group_by(GEOID, source, from_category) %>%
    mutate(row_share_pct = 100 * n_pixels / sum(n_pixels)) %>%
    ungroup()
}
```
This is the per-county version (keeps `GEOID` in both `group_by()` calls, so each plaintiff county gets its own independent row-normalized matrix). `compute_pooled_row_shares_subset()` is used for the plaintiff-pooled figure and `compute_pooled_row_shares()` is used for the all-counties figure, use the same logic with `GEOID` simply dropped from both `group_by()` calls, so pixel counts are summed across counties before the row-normalization happens. Hence, the pooled figures show what fraction of all pooled GM/Tolerant/Vulnerable pixels (across many counties) transitioned each way, not an average of each county's own percentage. `PERIODS` is a named list mapping a period label to either `NULL` (all years pooled) or a `c(year_from, year_to)` pair. `if (!is.null(period_years))` filters to just that one year-pair when a specific period is requested, and each function is called once per period via `purrr::map_dfr(names(PERIODS), ...)`, stacking the four periods' results into one table with a `period` column, cast to a factor with `levels = names(PERIODS)` to display in correct order (All years, 2015-2016, 2016-2017, 2017-2018).

### Building the annotated period labels

```r
vuln_to_gm_stats <- county_data %>%
  filter(from_category == "Vulnerable", to_category == "GM") %>%
  mutate(acres = n_pixels * PIXEL_ACRES) %>%
  select(period, source, acres) %>%
  pivot_wider(names_from = source, values_from = acres, values_fill = 0) %>%
  mutate(
    Baseline  = if ("Baseline"  %in% names(.)) Baseline  else 0,
    Corrected = if ("Corrected" %in% names(.)) Corrected else 0,
    diff_acres = Corrected - Baseline,
    diff_pct = if_else(Baseline == 0, NA_real_, 100 * diff_acres / Baseline)
  )

period_labels <- setNames(
  sprintf(
    "%s\n \nVulnerable to GM (ac)\nBaseline: %s\nCorrected: %s\nDiff: %s (%s)",
    as.character(vuln_to_gm_stats$period), ...
  ),
  as.character(vuln_to_gm_stats$period)
)
```
This block (repeated identically for the per-county, plaintiff-pooled, and all-counties versions) isolates the specific transition of interest (`Vulnerable to GM`), and converts its pixel count to acres (`n_pixels * PIXEL_ACRES`) for both sources. Then, it computes the absolute and percentage change from Baseline to Corrected. `pivot_wider(names_from = source, ...)` puts Baseline and Corrected side by side as columns. The two `if ("Baseline" %in% names(.)) ... else 0` is for cases where a period/subset combination has literally no data for one of the two sources (so `pivot_wider` wouldn't have created that column at all), defaulting it to 0 rather than erroring on a missing column reference. The resulting `Baseline: ... / Corrected: ... / Diff: ...` string (one per period) is folded directly into that period's label via `setNames()` (mapping period name to its annotated label string). Thus, the reader sees, right on the facet header, exactly how many acres of Vulnerable-to-GM transition the correction added or removed in that period, without needing a separate table alongside the matrix.

### The Matrix

```r
p_county <- ggplot(county_data, aes(x = to_category, y = from_category, fill = row_share_pct)) +
  geom_tile(color = "black", linewidth = 0.4) +
  geom_text(aes(label = sprintf("%.1f%%", row_share_pct)), color = TEXT_COLOR, size = 3, fontface = "bold") +
  scale_fill_gradient(low = FILL_LOW, high = FILL_HIGH, name = "% of\nrow total") +
  facet_grid(period ~ source, labeller = labeller(period = period_labels)) +
  ...
```
A standard `geom_tile()` heatmap with `geom_text()` printing the exact percentage on each cell, colored by `row_share_pct` on a fixed two-color gradient (`FILL_LOW`/`FILL_HIGH`. `facet_grid(period ~ source, ...)` is what produces the grid: rows are the 4 time periods, columns are Baseline vs. Corrected, so a reader can visually compare the same period's Baseline and Corrected 3×3 matrices side by side. `labeller(period = period_labels)` is what place correctly the acreage-annotated strings.

Section 4 (per-county) runs this same plot once per plaintiff county. Sections 5 and 6 reuse it for the plaintiff-pooled and all-counties figures, just without grouping by GEOID first, so counties are summed together before the row percentages are computed.

```r
missing_plaintiff <- setdiff(PLAINTIFF_GEOIDS, unique(tm_3x3_plaintiff$GEOID))
if (length(missing_plaintiff) > 0) {
  cat("WARNING -- missing from transition data entirely:", paste(missing_plaintiff, collapse = ", "), "\n")
}
```
A coverage check specific for `PLAINTIFF_GEOIDS` list. It flags, by name, any plaintiff county that has no transition data at all in the `common-counties-filtered` dataset.
