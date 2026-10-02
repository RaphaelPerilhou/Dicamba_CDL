# `14_classification_summary_final.R`

> Same per-county-year pixel-count-by-category summary as `01bis_classification_dataframes.R` see `readme01bis.md` for more details. Only what's different is covered below.

## Inputs
- `outputs/<VERSION>/final_corrected/<STATEFP>/Final_<YYYY>_<GEOID>.tif`: the best-model-corrected rasters from `13_apply_best_model.R`, instead of `01bis`'s raw disaggregated-CDL-code rasters.
- `data/county_lookup.csv`: as elsewhere.

## Outputs
- `outputs/<VERSION>/classification_summary_final_parts/<GEOID>.csv`: per-county checkpoint files.
- `outputs/<VERSION>/classification_summary_final.csv`:  combined long format.
- `outputs/<VERSION>/classification_summary_final_wide.csv`: combined wide format, one column per category plus a `total`.

## What's different from `01bis`

```r
r <- rast(in_path)
freq_tbl <- terra::freq(r)

if (nrow(freq_tbl) == 0) {
  cat("  Empty raster (no pixels), skipping:", in_path, "\n")
  next
}
```
Because `Final_*.tif` is already in aggregated category (0/1/2/3/99), we can directlty apply `terra::freq()` pixel count without doing the code-to-category mapping first. 

```r
wide <- long %>%
  pivot_wider(...) %>%
  mutate(total = NonCrop + GM + Tolerant + Vulnerable + Unclassified) %>%
  select(statefp, geoid, year, NonCrop, GM, Tolerant, Vulnerable, Unclassified, total)
```
Adds a `total` column (sum across all 5 categories) to the wide output, this is for downstream scripts that need each county-year's total pixel count without re-summing the category columns themselves.
