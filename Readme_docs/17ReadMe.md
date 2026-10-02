# `17_Census_validation_year.R`

## Inputs
- `data/qs.census2017.txt`: the 2017 USDA Census of Agriculture, same format as the 2012 file used throughout the rest of the pipeline.
- `data/metadata/<VERSION>/census_shortdesc_map.csv`: the `short_desc` to commodity mapping decided once during the 2012 analysis (`05bis_census_area.R`). It is reused unchanged here since.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `<VERSION>`): commodity-to-category mapping, as elsewhere.
- `outputs/<VERSION>/classified/<STATEFP>/Classified_2017_<GEOID>.tif`: the 2017 uncorrected classified raster.
- `outputs/<VERSION>/final_corrected/<STATEFP>/Final_2017_<GEOID>.tif`: the 2017 raster after applying each county's 2012-selected best model.
- `outputs/<VERSION>/post_processing/best_per_county_q1.csv`: each county's winning model and its 2012 bias-reduction figures (from `12_Post_processing_analysis.R`).
- `outputs/<VERSION>/category_census_acres_2012.csv`: 2012 county-level Census cropland acreage, used as the comparison point for the 2012-vs-2017 figure.
- `data/county_lookup.csv`, `data/SF/states_2016.rds`: as elsewhere.

## Outputs
- `data/CENSUS/<VERSION>/census_area_2017.csv`, `outputs/<VERSION>/category_census_acres_2017.csv`: the 2017 analogs of `05bis.R`'s and `06.R`'s outputs.
- `outputs/<VERSION>/bias_2017.csv`: long-format bias table, Baseline and Corrected stacked, for 2017.
- `outputs/<VERSION>/bias_2017_summary.csv`: one row per county: baseline vs. corrected total bias in 2017, and whether the 2012-selected model still improves things.
- Figures (`figures/<VERSION>/bias_2017_validation/`): `p_generalization_2012_vs_2017.png`, `p_distribution_by_model.png`, `p_impact_by_agri_size_2012_vs_2017.png`, `p_distrib_zeropct.png`, `p_overlap_0to5.png`, `p_switch_check.png`, `p_impact_by_agri_size_mmu_vs_csb.png`.
- Tables (`tables/<VERSION>/bias_2017_validation/`): `impact_by_agri_size_2012_vs_2017.csv`, `table_hurt_bins_baseline.tex`, `impact_by_agri_size_mmu_vs_csb.csv`, `table_impact_by_model.tex`.

## Summary
Every post-processing model in this pipeline was selected using 2012 data. This script checks if a county's fixed, 2012-selected model still reduce bias when checked against an independent year of Census data (2017) that had no part in choosing it. 

## Detailed explanation

```r
missing_descs_2017 <- setdiff(selected_descs, unique(area_stats_2017$short_desc))
if (length(missing_descs_2017) > 0) {
  cat("\nWARNING: the following short_desc labels used for 2012 were NOT found",
      "in the 2017 data -- check for renamed/restructured commodities:\n")
  print(missing_descs_2017)
}
```
Because `census_shortdesc_map.csv` was built once from the 2012 Census data and is reused here, this checks whether every `short_desc` label it relies on is still present in the 2017 extract (in case NASS renames or restructures how a commodity is reported between Census years).

```r
shortdesc_overrides_2017 <- tribble(
  ~commodity,        ~short_desc,
  "BERRIES, OTHER",  "BERRIES, OTHER - ACRES GROWN",
  "STRAWBERRIES",    "STRAWBERRIES - ACRES GROWN",
  "CRANBERRIES",     "CRANBERRIES - ACRES GROWN",
  "BLUEBERRIES",     "BLUEBERRIES - ACRES GROWN",
  "BEANS",           "BEANS, DRY EDIBLE, (EXCL CHICKPEAS & LIMA) - ACRES HARVESTED"
)

selected_descs_2017 <- c(
  setdiff(selected_descs, missing_descs_2017),
  shortdesc_overrides_2017$short_desc
)
```
Rather than dropping every commodity whose label changed (which would understate 2017 Census acreage for exactly those commodities), each renamed label found during the check above is manually added as an explicit substitution: berry-type crops moved from an "ACRES HARVESTED" to an "ACRES GROWN" statistic category between 2012 and 2017, Blueberries switched from separate Tame/Wild lines to one aggregate line, and Beans now exclude chickpeas. `selected_descs_2017` is the original 2012 label set with the renamed ones swapped out for their 2017 equivalents, so the same underlying commodities are captured in both years regardless of which label each year's Census used.


```r
tasks <- county_lookup %>%
  semi_join(best_assignment %>% filter(best_model != "Baseline"), by = "GEOID")
```
This only processes counties whose 2012-selected `best_model` is not `"Baseline"`.

```r
bias_2017_summary <- bias_2017 %>%
  group_by(GEOID, source) %>%
  summarise(total_abs_bias = sum(abs_delta_acres, na.rm = TRUE), .groups = "drop") %>%
  pivot_wider(names_from = source, values_from = total_abs_bias, names_prefix = "bias_2017_") %>%
  left_join(best_assignment %>% select(GEOID, best_model), by = "GEOID") %>%
  mutate(
    acres_change = bias_2017_Corrected - bias_2017_Baseline,  # negative = improve
    improved_2017 = acres_change < 0
  )
```
For each county, total absolute bias (summed across the 3 cropland categories) is computed once for Baseline and once for Corrected in 2017, then the two are pivoted side by side. `acres_change` is deliberately signed so that a negative value means bias went down (improvement) (the sign is flipped for readability (`acres_improved <- -acres_change`)).

```r
impact_category = case_when(
  is.na(pct_of_agri_size) ~ NA_character_,
  pct_of_agri_size >= 50 ~ "Improved massively (>=50%)",
  ...
  pct_of_agri_size > -5  ~ "Hurt slightly (0-5%)",
  ...
  TRUE                    ~ "Hurt massively (>=50%)"
)
```
`pct_of_agri_size` expresses `acres_improved` as a percentage of the county's total Census cropland acreage. This exact `case_when()`/`factor(levels = ...)` block is duplicated twice: once for the 2017 outcome and once for what the same model achieved in 2012, so both years have directly comparable bins.

```r
AGRI_SIZE_CUTOFF <- quantile(bias_2017_summary$total_acres_census_2017, probs = 0.10, na.rm = TRUE)
bias_2017_trimmed <- bias_2017_summary %>% filter(total_acres_census_2017 >= AGRI_SIZE_CUTOFF)
```
We trim bottom 10% acreage because `pct_of_agri_size`'s denominator makes the percentage unstable for the smallest counties.

```r
switch_table <- impact_2012 %>%
  select(GEOID, impact_category_2012 = impact_category) %>%
  inner_join(bias_2017_trimmed %>% select(GEOID, impact_category_2017 = impact_category), by = "GEOID") %>%
  filter(!is.na(impact_category_2012), !is.na(impact_category_2017)) %>%
  count(impact_category_2017, impact_category_2012) %>%
  group_by(impact_category_2017) %>%
  mutate(share_within_2017cat = 100 * n / sum(n)) %>%
  ungroup()
```
A transition table between 2012 and 2017 impact categories for the same counties. This tracks each county's own bin membership across years. `share_within_2017cat` is conditioned on the 2017 outcome ("of the counties that landed in impact category X in 2017, what share came from each 2012 impact category?").

```r
hurt_bins_full <- bias_2017_trimmed %>%
  filter(str_detect(impact_category, "^Hurt")) %>%
  left_join(
    read_csv(...) %>% select(GEOID, bias_base_2012 = bias_base),
    by = "GEOID"
  ) %>%
  group_by(impact_category) %>%
  summarise(
    n = n(),
    min_acreage = min(total_acres_census_2017, na.rm = TRUE),
    median_acreage = median(total_acres_census_2017, na.rm = TRUE),
    max_acreage = max(total_acres_census_2017, na.rm = TRUE),
    mean_baseline_2012 = mean(bias_base_2012, na.rm = TRUE),
    mean_baseline_2017 = mean(bias_2017_Baseline, na.rm = TRUE),
    .groups = "drop"
  )
```
Keeping only the counties where the fixed model hurt in 2017 (any `impact_category` starting with `"Hurt"`), this summarizes each severity bin's acreage range and the mean baseline bias in both years. Comparing `mean_baseline_2012` against `mean_baseline_2017` allows us to diagnostic if a "hurt" county's baseline bias was already small in 2012 and stayed small in 2017, a fixed absolute-acreage change from the model can easily flip the sign of a near 0 true effect.
