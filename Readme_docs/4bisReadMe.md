# `04bis_clip_confidence_disagg.R`

Same logic as  `03_stack_confidence.R`. The only difference is that this version stacks confidence onto the disaggregated (raw CDL code) raster instead of the classified (category) raster. See `4ReadMe.md` for the explanation, except for the three points below.

## Inputs
- `outputs/<VERSION>/classified_disagg/<STATEFP>/Disagg_<YYYY>_<GEOID>.tif`: the per-county, per-year raw CDL code raster (produced by `01_mask_and_keep_disagg.R`), in place of the classified category raster used by `03_stack_confidence.R`.
- `data/confidence_NAT/CDL_conf_2012_national_aligned.tif`: same national confidence raster as before.
- `data/SF/counties_2016.rds`, `data/county_lookup.csv`: same as before.

## Outputs
- `outputs/<VERSION>/class_and_conf_disagg/<STATEFP>/Conf_stacked_disagg_<YYYY>_<GEOID>.tif`: a two-band raster per county, band 1 named `"cdl_code"` (raw CDL code, unchanged from the disaggregated input) and band 2 named `"confidence"`. Under `class_and_conf_disagg/` instead of `class_and_conf/`.

## Summary
Identical purpose and mechanism to `03_stack_confidence.R`, clip the national confidence raster to each county (via the same `crop()` + `mask(touches = FALSE)` approach, pre-projecting all county boundaries once up front), verify geometry alignment with `compareGeom()`, and stack the two rasters into one two-band file.
## What's different, line by line
- `disagg <- rast(disagg_file)` reads from `DISAGG_PATH()` (the `classified_disagg` tree) instead of `CLASSIFIED_PATH()` (the `classified` tree).
- `if (!compareGeom(disagg, confidence, ...))` checks alignment against `disagg` instead of `classified`.
- `names(stacked) <- c("cdl_code", "confidence")` labels band 1 `"cdl_code"` instead of `"category"`, because this band holds raw CDL numeric codes, not category codes.

Everything else is exactly the same as in `03_stack_confidence.R`.
