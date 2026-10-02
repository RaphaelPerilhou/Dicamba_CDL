# `01_mask_disagg.R`

## Inputs
- `data/clipped/<STATEFP>/CDL_<year>_<GEOID>.tif`: the per-county, per-year clipped national CDL raster (produced by `00_setup_and_clip.R`), raw CDL numeric codes with `NA` outside the county boundary.
- `data/SF/states_2016.rds` / `data/SF/counties_2016.rds`: `sf` boundary objects (produced by `000_get_sf_boundaries.R`), used here only to derive the list of contiguous-US state FIPS codes and enumerate counties.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `"<VERSION>"`): the CDL-code-to-category mapping metadata. Only the `Keep`/`CDL Code` columns are used here to define which codes count as "agricultural" for masking.

## Outputs
- `outputs/<VERSION>/classified_disagg/<STATEFP>/Disagg_<year>_<GEOID>.tif`: one raster per county per year (2009–2020), holding the raw CDL code for every pixel that is inside that county's union agricultural mask (defined below) in ANY study year, and `NA` for every pixel never in the mask. Crucially, an in-mask pixel keeps its raw CDL code for that year even if that code isn't itself an agricultural one (e.g. Forest, code 63). No reclassification happens in this script.
- `outputs/<VERSION>/classification_summary_disagg_parts/<GEOID>.csv`: one CSV per county, with columns `statefp, geoid, year, cdl_code, n_pixels`. 

## Summary
This script serves as the core masking step and the actual producer of the disaggregated rasters that `01bis_classification_dataframes.R`, `01ter_classification_rasters.R`, `02_TM_disag.R`, and the confidence-stacking scripts all read from. For each county, it builds a union agricultural mask: a pixel is included if it carried an agricultural CDL code (per the metadata xslsx file `Keep == 1` list) inat least one of the 12 study years (2009–2020). It then applies that same union mask, unchanged, to every year's clipped CDL raster. Hence, the resulting disaggregated raster keeps each pixel's actual CDL code for that year (agricultural or not) wherever the pixel is inside the mask, and sets everything outside the mask to `NA`. This is deliberately unreclassified: the raw codes are preserved so that the aggregated 5-category scheme (NonCrop/GM/Tolerant/Vulnerable/Unclassified) can be recovered exactly downstream by mapping codes through the reclass table, while also keeping the granular per-code information available for anything that needs it (e.g. `02_TM_disag.R` for disaggregated transition matrices).

## Detailed explanation

```r
ag_codes <- read_excel(METADATA_XLSX, sheet = VERSION) %>%
  filter(Keep == 1) %>%
  pull(`CDL Code`) %>%
  sort()
```
Reads the metadata xlsx file and extracts the vector of CDL codes marked `Keep == 1`.

```r
make_mask <- function(year, geoid, statefp) {
  cdl <- rast(clipped_path(year, geoid, statefp))
  ifel(cdl %in% ag_codes, 1, NA)
}
```
For a single year, produces a binary mask: `1` where that year's CDL code is one of the agricultural `ag_codes`, `NA` everywhere else (including pixels that were already `NA` in the clipped raster, i.e. outside the county). `%in%` here is `terra` operator that works elementwise on raster values.

```r
make_union_mask <- function(years, geoid, statefp) {
  masks <- lapply(years, function(y) make_mask(y, geoid, statefp))
  
  union <- masks[[1]]
  for (i in seq_along(masks)[-1]) {
    union <- ifel(!is.na(union) | !is.na(masks[[i]]), 1, NA)
  }
  ...
  return(union)
}
```
Builds one single-year mask per year via `make_mask()`, then folds them together with a running logical OR. At each step, a pixel is kept in `union` (`1`) if it was already in `union` (i.e. agricultural in some earlier year) or it's agricultural in the current year's mask (`!is.na(masks[[i]])`). Otherwise, it becomes/stays `NA`. After looping through all years, `union` is `1` for any pixel that was agricultural in at least one of the 12 study years, and `NA` for a pixel that was never agricultural in any year. This is computed once per county (not once per county-year), since the same union mask is applied to every year afterward.

```r
keep_year <- function(year, geoid, statefp, union_mask) {
  out_path <- disagg_path(year, geoid, statefp)
  if (file.exists(out_path)) { return(out_path) }  # skip if already done

  cdl    <- rast(clipped_path(year, geoid, statefp))
  masked <- mask(cdl, union_mask)
  masked <- as.int(masked)
  ...
```
For a given year, applies the county's (already-computed) union mask to that year's CDL raster via `terra::mask()` (this sets to `NA` every pixel outside the union mask, while every pixel inside the mask keeps its own actual CDL code for that year).

```r
code_freq <- freq(masked) 
total <- sum(code_freq$count)
...
summary_path <- summary_part_path(geoid)
counts_rows <- data.frame(statefp = statefp, geoid = geoid, year = year,
                          cdl_code = code_freq$value, n_pixels = code_freq$count)

if (file.exists(summary_path)) {
  write.table(counts_rows, summary_path, append = TRUE,
              sep = ",", row.names = FALSE, col.names = FALSE)
} else {
  write.csv(counts_rows, summary_path, row.names = FALSE)
}

writeRaster(masked, out_path, overwrite = TRUE, datatype = "INT1U")
```
`terra::freq(masked)` tabulates how many pixels hold each distinct value in the masked raster (= pixel count for each CDL-code for that county-year). These counts are written to a per-county summary file (`summary_part_path(geoid)`), where the first year for a county creates the file with `write.csv()`, and every subsequent year for that same county appends rows with `write.table(..., append = TRUE, col.names = FALSE)`. We do this by county, because with counties processed in parallel across cluster workers, having every worker append to a single shared summary file could lead to simultaneous write which might cause some error, so more as a precautionary principle, we write one file per county and cnonsolidation later in `01bis_classification_dataframes.R`.

```r
run_county <- function(geoid, statefp, years) {
  missing <- years[!file.exists(sapply(years, clipped_path, geoid = geoid, statefp = statefp))]
  if (length(missing) > 0) {
    stop("Missing clipped files for GEOID ", geoid, " years: ", paste(missing, collapse = ", "),
         "\nRun 00_setup_and_clip.R first.")
  }
  
  all_done <- all(file.exists(sapply(years, disagg_path, geoid = geoid, statefp = statefp)))
  if (all_done) {
    return(invisible(NULL))
  }
  
  union_mask <- make_union_mask(years, geoid, statefp)
  paths <- lapply(years, function(y) keep_year(y, geoid, statefp, union_mask))
  return(paths)
}
```
`run_county()` is what runs for each county. It first verifies every year's clipped input actually exists. Then, it checks whether all of that county's disaggregated outputs already exist (if so, it skips the entire county immediately, without even building the union mask, allowing safe and fast resumability). If work remains, it builds the union mask once, then calls `keep_year()` for every year using that same mask.

```r
tasks <- counties_sf %>%
  sf::st_drop_geometry() %>%
  filter(STATEFP %in% TARGET_STATEFPS) %>%
  select(GEOID, STATEFP)

source("/softs/R/createCluster.R")
cl <- createCluster()

clusterExport(cl, c("counties_sf", "TARGET_YEARS",
                    "run_county", "make_union_mask", "make_mask",
                    "keep_year", "clipped_path", "disagg_path", "summary_part_path",
                    "ag_codes", "tasks"))

parLapplyLB(cl, seq_len(nrow(tasks)), function(i) {
  library(terra)
  library(dplyr)
  geoid   <- tasks$GEOID[i]
  statefp <- tasks$STATEFP[i]
  run_county(geoid, statefp, TARGET_YEARS)
})

stopCluster(cl)
```
Follows the same cluster-parallel pattern used throughout the project. One task per contiguous-US county, `parLapplyLB()` for dynamic load-balancing (since union-mask-building cost varies with county size). `clusterExport()` sends every function and lookup object each worker needs, and each worker reloads `terra`/`dplyr` before calling `run_county()` for its assigned county across all `TARGET_YEARS` at once. `stopCluster(cl)` releases the workers once every county is processed.
