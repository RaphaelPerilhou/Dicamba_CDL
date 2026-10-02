# `08_confidence_baseline.R`

## Inputs
- `outputs/<VERSION>/class_and_conf/<STATEFP>/Conf_stacked_<YYYY>_<GEOID>.tif`: per-county two-band raster (category + confidence), produced by `04_clip_confidence.R`.
- `outputs/<VERSION>/baseline_bias.csv`: county × category bias table (from `07_bias_measure.R`).
- `data/metadata/<VERSION>/missing_geoid_census.csv`, `data/county_lookup.csv`: as elsewhere.

## Outputs
- `outputs/<VERSION>/quality_baseline.csv`: one row per `(GEOID, Category)`, with pixel counts and mean CDL confidence, at both the category level and the whole-county cropland level.
- `outputs/<VERSION>/baseline_measures.csv`: `baseline_bias.csv` joined with the confidence measures above: one combined table of bias and confidence-quality metrics per county × category.

## Summary
This script builds the confidence-based "quality" baseline that later model/correction comparisons are measured against. For every county, it reads the stacked category+confidence raster and computes the mean CDL classification confidence among cropland pixels (GM/Tolerant/Vulnerable, excluding NonCrop/Unclassified), once for the whole county, and once per category, then merges these confidence figures onto the existing bias table.

## Detailed explanation

```r
cat_vals <- values(stacked[[1]])
conf_vals <- values(stacked[[2]])

is_cropland <- !is.na(cat_vals) & cat_vals %in% CROPLAND_CODES
mean_conf_county <- mean(conf_vals[is_cropland], na.rm = TRUE)
```
Pulls both bands' values into plain vectors and restricts to pixels that are (a) not `NA` (inside the union ag mask) and (b) actually one of the three cropland categories (`1,2,3`) excluding NonCrop (0) and Unclassified (99), since confidence in those categories isn't the concern here. `mean_conf_county` is the county-level quality baseline `Q_i`: the average CDL confidence across all cropland pixels regardless of category.

```r
cat_results <- lapply(CROPLAND_CODES, function(code) {
  is_cat <- !is.na(cat_vals) & cat_vals == code
  data.frame(GEOID = geoid, Category_Code = code, Category = CODE_TO_NAME[as.character(code)],
             n_pixels = sum(is_cat), mean_conf_category = mean(conf_vals[is_cat], na.rm = TRUE))
})
cat_df <- bind_rows(cat_results) %>% mutate(mean_conf_county = mean_conf_county)
```
Repeats the same mean-confidence calculation restricted to just one category at a time (`mean_conf_category`, `Q_i,c`). Each of the three per-category rows also carries the same `mean_conf_county` value, so both the county-wide and category-specific baselines sit side by side in one row, ready to be divided against each other later.

```r
# Q_i,c = mean_conf_category / mean_conf_county  (per category per county)
# Q_i   = 1 by construction at baseline (all cropland / all cropland)
# These denominators are needed when we later compute quality for a model:
# Q_model_i,c = mean_conf(reclassified pixels in category c) / mean_conf_category
# Q_model_i   = mean_conf(all reclassified pixels) / mean_conf_county
```
This comment is the key to why the script stops at raw means rather than computing the ratios itself: at baseline, `Q_i` (overall quality) is trivially `1` (all cropland compared to itself), so there's nothing informative to compute yet. What this script actually produces (`mean_conf_category` and `mean_conf_county`) are the denominators which will serve later for post-processing/model-comparison (once a correction model reclassifies some pixels, that script computes the new mean confidence among the reclassified pixels and divides it by these baseline means, to see whether the model corrects pixels with higher or lower confidence, in mean, than baseline). 

```r
baseline <- bias %>%
  select(GEOID, Category, n_suppressed, acres_cdl, acres_census, ..., weighted_accuracy_baseline) %>%
  left_join(
    quality_baseline %>% select(GEOID, Category, mean_conf_category, mean_conf_county),
    by = c("GEOID", "Category")
  )
write_csv(baseline, BASELINE_MEASURES_PATH)
```
Merges the confidence quality measures onto the existing bias table from `07_bias_measure.R`, giving `baseline_measures.csv` as a single reference table combining both dimensions (acreage bias vs. Census, and classification confidence) per county × category.
