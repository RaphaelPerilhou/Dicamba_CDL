# `07bis_bias_measure_disagg.R`

> Same bias metrics as `07_bias_measure.R`, computed at county × commodity granularity instead of county × category. See `7ReadMe.md` for what each metric means. Only what's different here is covered below.

## Inputs
- `outputs/<VERSION>/classification_summary_disagg.csv`: long-format per-CDL-code pixel counts (from `01bis`), instead of `07`'s already-category-aggregated `classification_summary.csv`.
- `outputs/<VERSION>/commodity_census_acres_2012.csv`: commodity-level  Census acreage (from `06_Census_Acres.R`).
- `outputs/<VERSION>/category_census_acres_2012.csv`: used only to get each county's total Census acreage across all 3 categories (for `delta_pct_total`'s denominator).
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx`, `data/metadata/<VERSION>/missing_geoid_census.csv`: same as `07`.

## Outputs
- `outputs/<VERSION>/baseline_bias_disagg_commodity.csv`: one row per `(GEOID, commodity)`, with CDL vs. Census acres and the same bias/accuracy metrics as `07`'s output.

## What's actually different from `07`

```r
cdl_commodity <- read_csv(SUMMARY_DISAG_PATH) %>%
  filter(year == 2012) %>%
  left_join(code_to_commodity, by = c("cdl_code" = "CDL Code")) %>%
  filter(!is.na(`Census Commodity`)) %>%
  group_by(GEOID, commodity = `Census Commodity`, Category) %>%
  summarise(pixels_cdl = sum(n_pixels), .groups = "drop")
```
Since the input here is raw per-CDL-code counts, this script first maps each code to its Census commodity (via the metadata excel) and sums pixel counts by commodity. `filter(!is.na(...))` drops CDL codes with no Census commodity at all (e.g. NonCrop codes), since there's nothing to compare them against here.

```r
fulldf <- cdl_commodity %>%
  filter(!GEOID %in% missing_geoid) %>%
  full_join(commodity_census, by = c("GEOID", "commodity")) %>%
  mutate(acres_cdl = replace_na(acres_cdl, 0), ...)
```
Uses `full_join()` instead of `07`'s `left_join()`: at commodity level, a commodity can appear on the Census side with zero CDL pixels for it.

```r
category_census_totals <- read_csv(CENSUS_CAT_PATH) %>%
  group_by(GEOID) %>%
  summarise(acres_census_total = sum(acres_census, na.rm = TRUE), .groups = "drop")
...
cdl_share_within_category = ifelse(acres_cdl_category_total > 0,
                                   acres_cdl / acres_cdl_category_total, NA_real_)
```
Two extra pieces of context not needed in `07`. (1) `category_census_total` (each county's category census total) and `cdl_share_within_category` letting a commodity's bias be read either against the whole county or against just its own category.
