# `15_Transition_final.R`

> Uses `terra::crosstab()` the same way as `02_TM_disag.R` to build a transition matrix between two years — see `readme02.md` for the crosstab basics. What's genuinely new here is the union-mask restriction and the `useNA = TRUE` argument. **Empirically, the resulting `NoData` category is dominated by masked-out background, not by genuine model failures** — see the note below the `crosstab()` explanation for why, and for the reason downstream analysis restricts these matrices to the 3×3 GM/Tolerant/Vulnerable submatrix rather than using `NoData` at all.

## Inputs
- `outputs/<VERSION>/final_corrected/<STATEFP>/Final_<YYYY>_<GEOID>.tif` — the best-model-corrected rasters (from `13_apply_best_model.R`), already in aggregated category space (0/1/2/3/99) — unlike `02`'s raw disaggregated CDL-code input, no crosswalk/collapse step is needed here.
- `outputs/<VERSION>/classified/<STATEFP>/Classified_<YYYY>_<GEOID>.tif` — used only as the reference raster carrying the county's union agricultural mask (from `01_mask_and_keep_disagg.R`); any single year works, since the union mask is identical across all years for a given county by construction.
- `data/county_lookup.csv` — as elsewhere.

## Outputs
- `outputs/<VERSION>/transition_final/<STATEFP>/TM_<YEAR_FROM><YEAR_TO>_<GEOID>.csv` — one 5(+NoData)×5(+NoData) category transition matrix per county per consecutive year pair (2009→2010, 2010→2011, ..., 2019→2020).

## Summary
Computes year-over-year category transition matrices on the final, model-corrected classification — the post-processing counterpart to `02_TM_disag.R`'s raw-CDL transition matrices. The script's own header comment states that masking both years to the union agricultural extent before tabulation, combined with `useNA = TRUE`, guarantees that any `NoData` cell in the output is genuine model-induced data loss (MMU's hole-filling not fully converging within `max_fill_iter` iterations), never off-mask background. **That reasoning turns out to be incorrect in practice.** A direct check on the actual `transition_final` output (sampling 30 county/year-pair CSVs) shows `NoData` accounts for a median of ~88% of all transitions (range ~21–99.9%), while `NonCrop` — the category that should reflect genuinely in-mask, non-agricultural pixels — stays under ~2% almost everywhere. See the explanation under the `crosstab()` call below for why: `useNA = TRUE` does not drop masked-out pixels the way the comment assumes, so the off-mask background ends up explicitly counted as `NoData → NoData` rather than excluded. This is the reason later analysis scripts restrict these matrices to the 3×3 GM/Tolerant/Vulnerable submatrix and disregard `NoData` (and `Unclassified`/`NonCrop`) entirely.

## Detailed explanation

```r
relabel_dimnames <- function(x) {
  ifelse(
    is.na(x) | x == "NA",
    "NoData",
    category_labels[as.character(x)]
  )
}
```
`terra::crosstab(..., useNA = TRUE)` returns a matrix whose row/column names are the raw pixel values as character strings, with an actual `NA` representing pixels with no data. `relabel_dimnames()` is applied to both the row and column names such that category codes get mapped through `category_labels` to their readable names. Anything that's `NA` or `"NA"` gets the explicit label `"NoData"` instead of being silently left as `NA` in the output CSV.

```r
union_mask_ref <- rast(ref_path)

r_from <- mask(rast(path_from), union_mask_ref)
r_to   <- mask(rast(path_to),   union_mask_ref)
```
`union_mask_ref` is simply one year's `Classified_*.tif`, serving as a source for the mask geometry. `mask()` re-applies that same footprint to both the `from` and `to` final rasters, so any pixel outside the county's agricultural mask becomes `NA` in both.

```r
tm      <- crosstab(stacked, useNA = TRUE)
```
 `useNA = TRUE` keeps `NA` as its own explicit category in the crosstab output rather than excluding it. Since `r_from` and `r_to` were just masked, every off-mask pixel is `NA` in both bands. Hence, with `useNA = TRUE`, `crosstab()` counts every one of those as a `"NoData" to "NoData"` transition instead of dropping it. We kept it for allowing the transition from "something" to "NA" as it could be meaningful, but it's hard to disentangle the NA out of mask from the NA because of model.

```r
if (!compareGeom(r_from, r_to, stopOnError = FALSE)) {
  cat("  Rasters not aligned for GEOID", geoid, "years", year_from, "-", year_to, "\n")
  return(NULL)
}
```
This is just verifying alignment (same pattern used in `03_confidence_nat.R` and `04_clip_confidence.R`) before stacking the two years together.
