# `02_TM_disag.R`

## Inputs
- `outputs/<VERSION>/classified_disagg/<STATEFP>/Disagg_<YYYY>_<GEOID>.tif`: one raster per county per year (produced by `01_mask_and_keep_disagg.R`), holding raw CDL numeric codes for every in-mask pixel and `NA` outside the union ag mask.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `"<VERSION>"`): the CDL-code-to-category mapping metadata. Columns used here: `Keep`, `CDL Code`, `Category Code`.

## Outputs
- `outputs/<VERSION>/transition_disagg/<STATEFP>/TMD_<year_from><year_to>_<GEOID>.csv`: the disaggregated (raw-CDL-code-level) transition matrix for one county and one consecutive year pair. A matrix whose dimensions vary by county/year-pair (only CDL codes actually present appear), with row names = CDL codes present in `year_from`, column names = CDL codes present in `year_to`, and cell values = pixel counts transitioning from that row's code to that column's code.
- `outputs/<VERSION>/transition/<STATEFP>/TM_<year_from><year_to>_<GEOID>.csv`: the aggregated 5x5 category-level transition matrix for the same county/year pair, collapsed from the disaggregated matrix above. Row/column names are the fixed category labels (`NonCrop, GM, Tolerant, Vulnerable, Unclassified`), cell values = pixel counts transitioning from that row's category to that column's category.

## Summary
This script computes, for every county and every pair of consecutive years (2009 to 2010, 2010 to 2011, ..., 2019 to 2020), a pixel-level transition matrix at the raw CDL-code level using `terra::crosstab()`, then collapses that disaggregated matrix into the fixed 5-category (NonCrop/GM/Tolerant/Vulnerable/Unclassified) transition matrix used everywhere else in the pipeline. As with the other cluster-parallel scripts, work is dispatched county-by-county across the ANUBIS cluster.

## Detailed explanation

```r
cdl_map <- read_excel(METADATA_XLSX, sheet = VERSION) %>%
  filter(Keep == 1)
code_to_category <- setNames(cdl_map$`Category Code`, as.character(cdl_map$`CDL Code`))
category_labels <- c("0" = "NonCrop", "1" = "GM", "2" = "Tolerant",
                     "3" = "Vulnerable", "99" = "Unclassified")
```
Same code-to-category lookup construction used in `01bis_classification_dataframes.R` and `01ter_classification_rasters.R`: a named vector mapping each `Keep == 1` CDL code (as a character key) to its numeric category code, plus a fixed label vector translating numeric categories to their names. Any CDL code absent from this map is treated as category `99` (Unclassified).

```r
compute_transition_disagg <- function(year_from, year_to, geoid, statefp, force = FALSE) {
  out_path <- TRANSITION_DISAGG_PATH(year_from, year_to, geoid, statefp)
  if (file.exists(out_path) && !force) {
    return(as.matrix(read.csv(out_path, row.names = 1, check.names = FALSE)))
  }
  ...
  r_from <- rast(path_from)
  r_to   <- rast(path_to)

  if (!compareGeom(r_from, r_to, stopOnError = FALSE)) {
    stop("Rasters are not aligned for GEOID ", geoid, " years ", year_from, "-", year_to)
  }

  stacked <- c(r_from, r_to)
  tm      <- crosstab(stacked)
  ...
  write.csv(tm, out_path)
  return(tm)
}
```
This is the core disaggregated-transition computation, one call per county per consecutive year pair. It is resumable like the other scripts (skips and re-reads from disk if the output already exists), and fails fast with `stop()` if either year's input raster is missing. `compareGeom(r_from, r_to, stopOnError = FALSE)` verifies the two rasters share the same extent, resolution, and CRS *without* throwing an error itself (`stopOnError = FALSE`), so the script can instead raise its own more informative error naming the GEOID and year pair if alignment fails. This steps is important because `crosstab()` requires pixel-for-pixel correspondence between the two layers to be meaningful.

`c(r_from, r_to)` stacks the two single-band rasters into one two-band `SpatRaster`. `terra::crosstab(stacked)` then cross-tabulates the two bands directly: for every pixel, it reads the value in band 1 (the `year_from` CDL code) and band 2 (the `year_to` CDL code), and counts how many pixels fall into each `(value_from, value_to)` combination. Hence, this is what produces the transition matrix in a single call, with `NA` pixels (outside the union mask in either year) automatically excluded from the tabulation. Row and column names of the result are the raw CDL codes actually observed as present in `year_from` / `year_to`, which is why the matrix's dimensions vary from one county/year-pair to the next (a county might have Peanuts (10) some years and never others, for instance). The matrix is written to CSV with `write.csv()`, keeping those row/column names.

```r
collapse_to_aggregated <- function(tmd, year_from, year_to, geoid, statefp) {
  out_path <- TRANSITION_AGG_PATH(year_from, year_to, geoid, statefp)
  if (file.exists(out_path)) {
    return(as.matrix(read.csv(out_path, row.names = 1, check.names = FALSE)))
  }

  map_codes <- function(codes) {
    cats <- code_to_category[codes]
    cats[is.na(cats)] <- 99
    category_labels[as.character(cats)]
  }

  row_cats <- map_codes(rownames(tmd))
  col_cats <- map_codes(colnames(tmd))

  collapsed <- rowsum(tmd, group = row_cats)
  collapsed <- t(rowsum(t(collapsed), group = col_cats))

  write.csv(collapsed, out_path)
  return(collapsed)
}
```
This is the step that derives the fixed 5x5 category-level matrix from the disaggregated one. `map_codes()` translates a vector of raw CDL codes (the row or column names of `tmd`) into their category labels: it looks up each code in `code_to_category`, defaults any unmapped code (`NA` result) to `99`, and converts the resulting numeric categories to their string labels via `category_labels`. This is applied once to the row names (`row_cats`, i.e. what category each `year_from` CDL code belongs to) and once to the column names (`col_cats`, same for `year_to`).

`rowsum(tmd, group = row_cats)` sums rows of `tmd` that share the same `row_cats` label into a single row (it's where the collapsing of multiple raw-code rows (e.g. all Tolerant) into one row per category happens), while columns remain at the raw-code level. `t(rowsum(t(collapsed), group = col_cats))` repeats the same collapsing operation on the columns: transposing turns columns into rows, `rowsum()` collapses them by `col_cats`, and transposing back restores the original row/column orientation. Hence, it's just collapsing the columns by category too. After both steps, `collapsed` is a matrix with (up to) 5 rows and 5 columns, one per category, with cell values equal to the total pixel count transitioning from that row's category to that column's category.

```r
tasks <- counties_sf %>%
   sf::st_drop_geometry() %>%
   filter(STATEFP %in% TARGET_STATEFPS) %>%
   select(GEOID, STATEFP)

source("/softs/R/createCluster.R")
cl <- createCluster()

clusterExport(cl, c("counties_sf", "TARGET_YEARS",
                     "compute_transition_disagg", "collapse_to_aggregated",
                     "DISAGG_PATH", "TRANSITION_DISAGG_PATH",
                     "TRANSITION_AGG_PATH", "code_to_category",
                     "category_labels", "tasks"))

parLapplyLB(cl, seq_len(nrow(tasks)), function(i) {
   library(terra)
   library(dplyr)
   geoid   <- tasks$GEOID[i]
   statefp <- tasks$STATEFP[i]
   for (j in seq_len(length(TARGET_YEARS) - 1)) {
     tmd <- compute_transition_disagg(TARGET_YEARS[j], TARGET_YEARS[j+1], geoid, statefp)
     collapse_to_aggregated(tmd, TARGET_YEARS[j], TARGET_YEARS[j+1], geoid, statefp)
   }
})

stopCluster(cl)
```
Follows the same cluster-parallel pattern as the other ANUBIS scripts in this pipeline: one task per contiguous-US county, dispatched with `parLapplyLB()` for dynamic load-balancing across workers (since counties differ greatly in raster size/processing cost). The inner loop, `seq_len(length(TARGET_YEARS) - 1)`, walks through all 11 consecutive year pairs within `TARGET_YEARS` (2009–2020). `j` and `j+1` index adjacent years, so this computes 2009 to 2010, 2010 to 2011, ..., 2019 to 2020 for each county, calling `compute_transition_disagg()` and then immediately `collapse_to_aggregated()` on its result for each pair. `clusterExport()` sends every function and lookup object each worker needs (both computation functions, both path builders, the category mapping, and the task list) into the workers' environments so each worker reloads `terra` and `dplyr` before starting its loop. `stopCluster(cl)` stops the worker processes once all counties and year-pairs are done.
