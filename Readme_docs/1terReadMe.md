# `01ter_classification_rasters.R`

## Inputs
- `outputs/<VERSION>/classified_disagg/<STATEFP>/Disagg_<YYYY>_<GEOID>.tif`: one raster per county per year (produced upstream, by `01_mask_and_keep_disagg.R`), holding raw CDL numeric codes for every in-mask pixel and `NA` for every pixel outside the union ag mask.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `"<VERSION>"`): the CDL-code-to-category mapping metadata. Columns used here: `Keep`, `CDL Code`, `Category Code`.

## Outputs
- `outputs/<VERSION>/classified/<STATEFP>/Classified_<YYYY>_<GEOID>.tif`: one raster per county per year, same extent/resolution/mask footprint as the input, but with pixel values recoded from raw CDL codes into the category scheme: `0` = NonCrop, `1` = GM, `2` = Tolerant, `3` = Vulnerable, `99` = Unclassified, `NA` = outside the union ag mask. 

## Summary
This script converts the per-county, per-year disaggregated CDL rasters (raw numeric CDL codes) into category-level classified rasters (NonCrop/GM/Tolerant/Vulnerable/Unclassified), by applying the CDL-code to category mapping from the metadata .xslx file as a raster reclassification. It is the raster-level counterpart to `01bis_classification_dataframes.R`. It produces the actual reclassified raster files, used by any downstream step that needs per-pixel category rasters (transition matrices, post-processing, etc.), not just aggregate counts. Like `00_setup_and_clip.R`, the actual reclassification work is parallelized county-by-county across the ANUBIS cluster.

## Detailed explanation

```r
cdl_map <- read_excel(METADATA_XLSX, sheet = VERSION) %>%
  filter(Keep == 1)

reclass_table <- as.matrix(cdl_map[, c("CDL Code", "Category Code")])
dimnames(reclass_table) <- NULL
```
Reads the metadata .xlsx's `v1`/`v2` sheet (matching `VERSION`) and keeps only `Keep == 1` rows (the CDL codes actually included in the analysis). `reclass_table` is then built as a plain two-column numeric matrix (`CDL Code`, `Category Code`), with `dimnames` stripped, because this is exactly the input format `terra::classify()` expects: a two-column "from, to" lookup matrix used to remap raster values.

```r
classify_from_disagg <- function(year, geoid, statefp) {
  out_path <- CLASSIFIED_PATH(year, geoid, statefp)
  if (file.exists(out_path)) { return(out_path) }  # skip if already done

  in_path <- DISAGG_PATH(year, geoid, statefp)
  if (!file.exists(in_path)) {
    stop("Missing disaggregated raster: ", in_path, "\nRun 01_mask_and_keep_disagg.R first.")
  }

  disagg <- rast(in_path)
  classified <- classify(disagg, reclass_table, others = NA)
  ...
```
`classify_from_disagg()` is the core per-county-per-year operation, structured the same resumable way as the clipping step in `00_setup_and_clip.R`: it skips immediately if the output already exists, and fails loudly with `stop()` (naming the missing file and the upstream script to rerun) if the required input raster is missing, rather than silently producing a wrong or empty output. `terra::classify(disagg, reclass_table, others = NA)` performs the actual reclassification: every pixel whose value matches a "from" value in `reclass_table` is remapped to its corresponding category, and every pixel whose value does not appear in `reclass_table` (including pixels that were already `NA` in `disagg`) is set to `NA` by the `others = NA` argument. At this point, `classified` cannot yet distinguish between two different kinds of `NA`: "outside the union ag mask" (already `NA` in `disagg`) and "in-mask but an unmapped/excluded CDL code" (a real CDL value that just isn't in `reclass_table`).

```r
  classified <- ifel(
    !is.na(disagg) & is.na(classified), 99L,
    classified
  )
  classified <- as.int(classified)
```
This step resolves that ambiguity. `ifel(!is.na(disagg) & is.na(classified), 99L, classified)` checks, pixel by pixel. If a pixel is not `NA` in the original disaggregated raster (i.e. it did carry some real CDL code, so it's inside the union ag mask), but is `NA` after classification (meaning that CDL code wasn't in `reclass_table`)? If so, it's recoded to `99` (Unclassified) (any in-mask pixel with an unmapped or excluded CDL code falls through to Unclassified, rather than being lost as `NA`). Every other pixel keeps its already-computed `classified` value: either a real category (0/1/2/3) if it matched the reclass table, or `NA` if it was genuinely outside the mask.

```r
writeRaster(classified, out_path, overwrite = TRUE, datatype = "INT1U")
```
Writes the final classified raster.

```r
tasks <- counties_sf %>%
  sf::st_drop_geometry() %>%
  filter(STATEFP %in% TARGET_STATEFPS) %>%
  select(GEOID, STATEFP)

source("/softs/R/createCluster.R")
cl <- createCluster()

clusterExport(cl, c("tasks", "TARGET_YEARS",
                    "classify_from_disagg", "CLASSIFIED_PATH", "DISAGG_PATH",
                    "reclass_table", "category_labels"))

parLapplyLB(cl, seq_len(nrow(tasks)), function(i) {
  library(terra)
  geoid   <- tasks$GEOID[i]
  statefp <- tasks$STATEFP[i]
  for (year in TARGET_YEARS) {
    classify_from_disagg(year, geoid, statefp)
  }
})

stopCluster(cl)
```
Parallelization follows the same pattern as `00_setup_and_clip.R`: one task per contiguous-US county, each worker looping over all `TARGET_YEARS` (2009–2020) for its assigned county. `createCluster()` (from the ANUBIS-specific cluster setup script) launches worker processes. `clusterExport()` copies the objects and functions each worker needs (including the pre-built `reclass_table`, computed once in the main process rather than re-read from the Excel file by every worker) into their environments. `parLapplyLB()` load-balances county assignments dynamically across workers as they finish, since counties vary widely in raster size/processing cost. Each worker reloads `terra` (required since it isn't inherited from the parent process) before calling `classify_from_disagg()` for each year of its assigned county. `stopCluster(cl)` shuts the workers down once every county-year has been processed.
