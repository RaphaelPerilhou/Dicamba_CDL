# `07_bias_measure.R`

## Inputs
- `outputs/<VERSION>/category_census_acres_2012.csv`: category-level Census acreage per county (produced by `06_Census_Acres.R`).
- `outputs/<VERSION>/classification_summary.csv`: category-level CDL pixel counts per county-year (produced by `01bis_classification_dataframes.R`); only `year == 2012` is used.
- `data/metadata/<VERSION>/missing_geoid_census.csv`: counties absent from the Census entirely (produced by `05bis`).

## Outputs
- `outputs/<VERSION>/baseline_bias.csv`: one row per `(GEOID, Category)` for GM/Tolerant/Vulnerable, with CDL acres, Census acres, and several bias/accuracy metrics comparing them.
- `data/metadata/<VERSION>/zero_census_acres_geoids.csv`: the list of counties where Census reports zero acres across all three categories, flagged for downstream awareness rather than excluded.

## Summary
This is the core bias-measurement script. It builds the county × category table comparing CDL-derived acreage against Census acreage for 2012, computing several bias metrics (raw difference, percent relative to Census, percent relative to CDL, percent relative to the county's total Census acreage, and an "accuracy" ratio), and is what all the later post-processing/correction scripts ultimately measure improvement against.

## Detailed explanation

```r
classification_summary %>% count(GEOID, name = "n_row_counties") %>% count(n_row_counties, name = "n_counties")
census_acres %>% count(GEOID, name = "n_row_counties") %>% count(n_row_counties, name = "n_counties")
```
Both are quick diagnostic checks (not saved) on whether every county has exactly 3 rows (one per category). CDL always does, since `classification_summary.csv` is built with `values_fill = 0`. However, Census has no row for a county-category where zero commodity is reported.

```r
fulldf <- classification_summary %>%
  select(GEOID, Category, pixels_cdl, acres_cdl) %>%
  filter(!GEOID %in% missing_geoid) %>%
  left_join(census_acres, by = c("GEOID", "Category"))

fulldf <- fulldf %>%
  mutate(acres_census = replace_na(acres_census, 0),
         n_suppressed = replace_na(n_suppressed, 0))
```
Starts from the CDL side (complete) and left-joins Census acreage onto it, after dropping counties with no Census presence at all (`missing_geoid`, from `05bis`). Any `(GEOID, Category)` combination absent from Census, get `NA` from the join and is explicitly replaced by `0`, restoring the 3 rows per county.

```r
delta_pct_census = case_when(
  acres_census == 0 & acres_cdl == 0 ~ 0,
  acres_census == 0 & acres_cdl != 0 ~ NA_real_,
  TRUE ~ (acres_cdl - acres_census) / acres_census
),
delta_pct_CDL = case_when(
  acres_census == 0 & acres_cdl == 0 ~ 0,
  acres_census != 0 & acres_cdl == 0 ~ NA_real_,
  TRUE ~ (acres_cdl - acres_census) / acres_cdl
),
```
Two versions of relative bias, one normalized by Census, one by CDL. We use two measures as there is few cases where both CDL and Census are 0, meaning most of the time, at least one of the two measure is defined. Moreover, keeping both denominators (Census and CDL) rather than picking one let us the flexibility to decide later which measure to use.

```r
acres_census_total = sum(acres_census),
delta_pct_total = (acres_cdl - acres_census) / acres_census_total,

acres_cdl_total = sum(acres_cdl),
cdl_share = acres_cdl / acres_cdl_total,
accuracy_baseline = ifelse(
  acres_cdl == 0 & acres_census == 0, 1,
  pmin(acres_cdl, acres_census) / pmax(acres_cdl, acres_census)
),
weighted_accuracy_baseline = accuracy_baseline * cdl_share
```
All computed `group_by(GEOID)`, so `acres_census_total`/`acres_cdl_total` are each county's 3-category sum. `delta_pct_total` expresses a category's bias as a share of the county's total Census acreage, rather than that category's own. This is very useful when a category's own denominator is small or zero but the county overall has substantial acreage. `accuracy_baseline` is an aggreement ratio (`min/max`) of the two values (equals 1 for a perfect match and shrinks toward 0 as the two diverge), we explicitly define it as = 1 when both are zero (rather than the undefined `0/0`). `cdl_share` weights each category by its share of the county's total CDL acreage, so `weighted_accuracy_baseline` summed across a county's 3 categories gives a single county-level accuracy score dominated by whichever category actually has the most acreage.

```r
bias %>% filter(is.na(cdl_share))
#County concerned: GEOID = 27031
```
`cdl_share` is `NA` only when `acres_cdl_total == 0`. We found only a single county (27031) concerned with negligible Census acreage too. Hence, we judge irrelevant and simply excluded from the informal `county_level` accuracy check (not from `baseline_bias.csv` itself, which retains it).

```r
bias <- bias %>%
  group_by(GEOID) %>%
  mutate(zero_census_total = sum(acres_census, na.rm = TRUE) == 0) %>%
  ungroup()

zero_census_counties <- bias %>% filter(zero_census_total) %>% distinct(GEOID)
write_csv(zero_census_counties, ZERO_CENSUS_PATH)
```
Flags counties where Census reports zero acreage across all three categories combined (20 counties).
