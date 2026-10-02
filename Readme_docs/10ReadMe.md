# `100_Reclass_X.R`

> **Note on the 11 copies:** this script exists as 11 near-identical files (`100_Reclass_0.R` to `100_Reclass_10.R`). They differ ONLY in the `WHICHPART` value. This is purely so each copy can be submitted to a different ANUBIS node/job at the same time to gain time running only a subset of all counties per nodes, and then merging. Hence, a single script re-run with `WHICHPART` changed each time would work identically. 

## Inputs
- `outputs/<VERSION>/class_and_conf/<STATEFP>/Conf_stacked_<YEAR>_<GEOID>.tif`: per-county category+confidence raster (from `04_clip_confidence.R`).
- `outputs/<VERSION>/baseline_measures.csv`: county × category bias and confidence baseline (from `08_confidence_baseline.R`).
- `data/county_lookup.csv`, `data/SF/counties_2016.rds`: county reference and boundaries.
- `data/NationalCSB_2013-2020_rev23/CSB1320.gdb` (layer `national1320`): the national Common Sequence Boundaries (CSB) field-polygon dataset from Hunt et al. (2024), used by the CSB correction model.

## Outputs (all under a `parts/part_<WHICHPART>/` subfolder, so each parallel chunk writes independently)
- `outputs/<VERSION>/parts/part_<N>/mmu_by_county/<GEOID>.rds`: one checkpoint per county, that county's full MMU parameter-grid results (18 combinations) once every one has succeeded.
- `outputs/<VERSION>/parts/part_<N>/csb_by_county/<GEOID>.rds`: one checkpoint per county for the CSB model.
- `outputs/<VERSION>/parts/part_<N>/models_processing_details.csv`, every checkpoint combined: one row per `(GEOID, model)`, acreage bias before/after correction and confidence-quality metrics.
- `outputs/<VERSION>/parts/part_<N>/models_processing_summary.csv`: collapsed to one row per `(GEOID, model)` with total bias, bias-improvement share, and quality ratio.
- `figures/<VERSION>/parts/part_<N>/post_processing/<STATEFP>_<GEOID>_<model_name>.png`: a diagnostic map of each model's corrected raster, if `MAKE_PLOTS = TRUE`.

## Methodology and code: MMU + spatial filtering (`run_mmu_model`)

Following Lark et al. (2017): a contiguous patch of a crop category smaller than a chosen area threshold is treated as classification noise, removed, and refilled with the modal category among its surrounding pixels. We argue that the surrounding classification is more reliable than an implausibly small, isolated patch.

```r
mmu_model_grid <- tribble(
  ~mmu_acres, ~neighbors, ~window, ~max_fill_iter,
  35,          8,          3,       3,
  35,          8,          9,       3,
  ...
  1,           8,          3,       1
) %>%
  mutate(model_name = paste0("MMU_", mmu_acres, "ac_nb", neighbors, "_w", window, "_iter", max_fill_iter))
```
Since the right threshold/window/neighbor rule combination isn't known (and arguably depends on each county as they are heterogeneous), this grid runs 18 parameter combinations (`mmu_acres` from 1 to 35 acres, `neighbors` = 4 or 8, `window` = 3 or 9), each scored independently against Census afterward.

```r
run_mmu_model <- function(original_category, mmu_acres, neighbors, window, max_fill_iter,
                          model_name, cropland_codes = CROPLAND_CODES, pixel_acres = PIXEL_ACRES) {
  output <- terra::deepcopy(original_category)

  for (code in cropland_codes) {
    class_raster <- ifel(original_category == code, 1, NA)
    patch_raster <- terra::patches(class_raster, directions = neighbors, zeroAsNA = TRUE)
```
For each cropland code (1, 2, 3) in turn, every other code is set to `NA` so that `terra::patches()` can identify contiguous patches of that code alone (`patches()` only recognizes a patch as a group of cells surrounded by `NA`), so isolating one code at a time is the only way to detect a patch of code `A` sitting inside a field of code `B`. The `directions` parameter (`neighbors`, 4 or 8) sets whether diagonal neighbors count as contiguous (Queen's case, 8) or not (Rook's case, 4).

```r
    freq_tbl  <- terra::freq(patch_raster)
    small_ids <- freq_tbl$value[freq_tbl$count * pixel_acres < mmu_acres]
    if (length(small_ids) == 0) next

    temp <- ifel(patch_raster %in% small_ids, NA, original_category)
```
`terra::freq()` counts each patch's pixels. Any patch whose area (pixels × `pixel_acres`) falls below `mmu_acres` is flagged as noise (`small_ids`). `temp` then holds the original classification everywhere except those small patches, which become `NA` . Hence, at this point every pixel is either `NA` (a small patch, or already outside the mask) or its original value. Hence, as showed below we distinguish between the two types of `NAs`.

```r
    for (iter in 1:max_fill_iter) {
      holes   <- is.na(temp) & !is.na(original_category)
      n_holes <- global(holes, "sum", na.rm = TRUE)[1, 1]
      if (n_holes == 0) break
      temp <- terra::focal(temp, w = window, fun = "modal", na.rm = TRUE, na.policy = "only")
    }
    fill <- temp
    output <- ifel(patch_raster %in% small_ids, fill, output)
  }
```
`holes` are the pixels that are `NA` in `temp` but were not `NA` in `original_category` (i.e. exactly the small patches to refill). `terra::focal()` runs a `window`×`window` moving-window modal filter (`na.policy = "only"` restricts changes to `NA` pixels), replacing each hole with the most common value among its non-`NA` neighbors. Because `focal()`'s internal processing order isn't controlled, a single pass may not close every hole, as a precaution, this repeats up to `max_fill_iter` times, breaking early once `n_holes == 0`. `output` is then updated with the refilled patches for this code only, leaving every other code's pixels untouched, before moving to the next `code`.

```r
  model_raster <- ifel(is.na(original_category), NA, output)
  freq_tbl <- terra::freq(model_raster)
  freq_tbl <- freq_tbl[freq_tbl$value %in% cropland_codes, ]
  model_acres <- tibble(
    Category        = unname(CODE_TO_NAME[as.character(freq_tbl$value)]),
    acres_cdl_model = freq_tbl$count * pixel_acres
  )
  list(raster = model_raster, acres = model_acres, model_name = model_name)
}
```
`model_raster` re-masks `output` to the original raster's valid extent (a robustness safeguard as `output` should already share the same `NA` pattern by construction). Category-level acreage is then computed exactly as for the uncorrected baseline: `terra::freq()` counts pixels per value, restricted to the three cropland categories, converted to acres via `pixel_acres`. The function returns the corrected raster, its acreage, and `model_name` (= the label used to compare this specific parameter combination against every other one, and against CSB, later).

## Methodology and code: CSB field-modal (`run_csb_model`)

A more exploratory alternative using Common Sequence Boundaries (field polygons delineated by Hunt et al. (2024) from multi-year crop-rotation patterns, on the reasoning that pixels within the same physical field tend to share a rotation pattern across years even if individual years misclassify some of them). Rather than correcting small isolated patches, this reclassifies every pixel within a field polygon to that field's modal CDL category, treating the field boundary itself as the unit of correction. (Caveat: these boundaries and rotations are external to this study's own classification and period, so building field boundaries directly from this project's own data could improve on this further.)

```r
run_csb_model <- function(original_category, county_poly, csb_state,
                          cropland_codes = CROPLAND_CODES, pixel_acres = PIXEL_ACRES) {
  county_poly_proj <- st_transform(county_poly, st_crs(csb_state))
  csb_county <- st_filter(csb_state, county_poly_proj)
  if (nrow(csb_county) == 0) return(NULL)
```
The county polygon is reprojected to match the (already-loaded, state-level) CSB layer's CRS; `st_filter()` keeps only CSB field polygons that intersect the county. If none intersect, there's nothing to correct and the function returns `NULL` immediately.

```r
  csb_vect   <- vect(st_transform(csb_county, crs(original_category)))
  modal_cats <- terra::extract(original_category, csb_vect, fun = "modal", na.rm = TRUE)
  csb_county$modal_cat <- as.integer(modal_cats[[names(original_category)]])
  csb_county$modal_cat[is.na(csb_county$modal_cat)] <- 99L
```
Reprojects again (this time to the raster's own CRS) and converts to a `terra` vector for use with `terra::extract()`, which computes the modal CDL category among all pixels falling inside each field polygon. Any field for which no mode could be computed is set to `99` (Unclassified) rather than left `NA`.

```r
  csb_vect_fb <- vect(st_transform(csb_county, crs(original_category)))
  model_raster <- rasterize(csb_vect_fb, original_category, field = "modal_cat")
  model_raster <- mask(model_raster, original_category)
  model_raster <- ifel(!is.na(original_category) & is.na(model_raster), original_category, model_raster)
```
`rasterize()` transfers each field's `modal_cat` back onto a raster matching `original_category`'s grid, so every pixel takes on its containing field's modal value. `mask()` forces pixels outside the agricultural mask back to `NA`. Because CSB fields may not cover every in-mask pixel, any pixel valid in `original_category` but left unset by `rasterize()` (still `NA`) reverts to its original classification. Hence, the correction only actually changes pixels genuinely covered by a field boundary.

```r
  freq_tbl <- terra::freq(model_raster)
  freq_tbl <- freq_tbl[freq_tbl$value %in% cropland_codes, ]
  model_acres <- tibble(
    Category        = unname(CODE_TO_NAME[as.character(freq_tbl$value)]),
    acres_cdl_model = freq_tbl$count * pixel_acres
  )
  list(raster = model_raster, acres = model_acres, model_name = "CSB_field_modal")
}
```
Same acreage computation and return structure as `run_mmu_model()`, but with a fixed model name (`"CSB_field_modal"`) rather than one varying across a parameter grid, since this method has no tenable parameters.

## Scoring a model against Census (`score_model`, `compute_quality`)

```r
score_model <- function(model_output, stacked, baseline, geoid, statefp, make_plots) {
  if (is.null(model_output)) return(list(results = NULL, quality = NULL))
  if (make_plots) plot_model(model_output$raster, geoid, statefp, model_output$model_name)

  results <- baseline %>%
    left_join(model_output$acres, by = "Category") %>%
    mutate(
      acres_cdl_model   = replace_na(acres_cdl_model, 0),
      delta_acres_model = acres_cdl_model - acres_census,
      improvement_acres = abs(delta_acres_baseline) - abs(delta_acres_model),
      improvement_pct   = 100 * (abs(delta_acres_baseline) - abs(delta_acres_model)) / abs(delta_acres_baseline),
      model = model_output$model_name
    )

  quality <- compute_quality(stacked, model_output$raster, geoid, model_output$model_name)
  list(results = results, quality = quality)
}
```
Joins the corrected model's category acreage onto the county's existing baseline bias row and computes `improvement_acres`: how much closer (in absolute acres) the corrected classification landed to Census than the uncorrected baseline did. A positive value means the model reduced bias for that category, while a negative means it made it worse. `improvement_pct` expresses the same thing relative to the baseline's own bias magnitude.

```r
compute_quality <- function(stacked_raster, model_category_raster, geoid, model_name) {
  orig_cat <- stacked_raster[["category"]]
  conf     <- stacked_raster[["confidence"]]
 
  reclass_ind <- ifel(!is.na(orig_cat) & !is.na(model_category_raster) &
                        orig_cat != model_category_raster, 1L, 0L)
  crop_ind    <- ifel(orig_cat %in% CROPLAND_CODES, 1L, 0L)
 
  z_re <- zonal(conf, reclass_ind, fun = "mean", na.rm = TRUE)
  z_cr <- zonal(conf, crop_ind,    fun = "mean", na.rm = TRUE)
 
  mean_conf_reclassified <- if (1 %in% z_re[[1]]) z_re[z_re[[1]] == 1, 2] else NA_real_
  mean_conf_all_cropland <- if (1 %in% z_cr[[1]]) z_cr[z_cr[[1]] == 1, 2] else NA_real_
  ...
  quality_ratio = mean_conf_reclassified / mean_conf_all_cropland
```
A model could reduce Census bias while, by coincidence, changing pixels the original classification was actually confident about. The quality checks for that. `reclass_ind` flags every pixel that change category because of the model. `crop_ind` flags every original cropland pixel. `zonal(conf, reclass_ind, "mean")` returns a small table with one row per group value (0 = unchanged, 1 = reclassified) and the mean confidence within that group, and `z_cr` works the same way over the crop/non-crop split. Because `zonal()` only returns rows for group values that actually occur in the raster, a county where the model reclassified nothing would have no `1` row in `z_re` at all. Therefore, `if (1 %in% z_re[[1]]) z_re[z_re[[1]] == 1, 2] else NA_real_` checks whether that row exists before pulling the confidence value out of it. Puts a `NA` instead of erroring on a missing row (the same guard is applied to `z_cr`, for the case of a county with no cropland pixels). `mean_conf_reclassified` is thus the mean CDL confidence among just the reclassified pixels, and `mean_conf_all_cropland` the mean among all cropland pixels. Dividing the two gives `quality_ratio`, where a value below 1 means the pixels this model chose to reclassify were, on average, less confidently classified to begin with, which is a good sign, since it suggests the model is correcting more uncertain pixels.


## Resumability and parallelism

```r
if (WHICHPART == 0) {
  tasks <- county_lookup %>% slice_head(n = N_TEST_COUNTIES)
} else {
  part_index <- cut(seq_len(nrow(county_lookup)), breaks = N_PARTS, labels = FALSE)
  tasks <- county_lookup[part_index == WHICHPART, ]
}
```
`county_lookup` (~3000 counties) is split into `N_PARTS` roughly equal chunks via `cut()`, and each script copy processes only the chunk matching its own hardcoded `WHICHPART`. This allow the full national run to be spread across multiple ANUBIS nodes at once. `WHICHPART = 0` is a small test mode (`N_TEST_COUNTIES` counties) for a quick sanity check before committing a full node run.

```r
process_county_mmu <- function(geoid, statefp, mmu_model_grid, baseline_all, make_plots = FALSE) {
  out_path <- mmu_county_path(geoid)
  if (file.exists(out_path)) { return(invisible(NULL)) }  # already done
  ...
  mmu_failed <- FALSE
  for (i in seq_len(nrow(mmu_model_grid))) {
    model_output <- tryCatch(run_mmu_model(...), error = function(e) { ...; NULL })
    if (is.null(model_output)) { mmu_failed <- TRUE; next }
    scored <- score_model(model_output, stacked, baseline, geoid, statefp, make_plots)
    county_results[[row$model_name]]  <- scored$results
    quality_results[[row$model_name]] <- scored$quality
  }
  if (mmu_failed) {
    return(invisible(NULL))  # NOT checkpointed — full county retried next run
  }
  saveRDS(list(results = bind_rows(county_results), quality = bind_rows(quality_results)), out_path)
}
```
Checks for an existing checkpoint first and skips immediately if found to avoid  re-reading raster, and useless computations. Each of the 18 grid rows is wrapped in `tryCatch()` so one bad parameter combination doesn't crash the whole county, but very IMPORTANTLY, the checkpoint is only written once every grid row has succeeded (`mmu_failed` tracks this). If any row failed, the function returns without saving anything, so the entire county's grid is retried from scratch on the next run (this is to avoid one parameter combination to fail silently and later forget to reapply it).  Hence, the recovery is simple, we just need to resubmit the same job with the same `WHICHPART` picks up exactly where it left off, since every already-completed county is skipped and nothing is left half-done on disk.

```r
process_state_csb <- function(statefp, geoids_in_state, baseline_all, counties_sf, make_plots = FALSE) {
  remaining_geoids <- geoids_in_state[!file.exists(csb_county_path(geoids_in_state))]
  if (length(remaining_geoids) == 0) return(invisible(NULL))  # skip state entirely

  csb_state <- st_read(CSB_GDB, layer = CSB_LAYER,
                       query = paste0("SELECT * FROM ", CSB_LAYER, " WHERE CSBID LIKE '", statefp, "%'"),
                       quiet = TRUE)

  invisible(lapply(remaining_geoids, function(geoid) {
    ...
    saveRDS(scored, csb_county_path(geoid))
  }))
}
```
CSB is parallelized per state, not per county, because reading a state's CSB polygons from the geodatabase (`st_read(..., query = "... WHERE CSBID LIKE '<statefp>%'")`) is the expensive part of this model. Doing it once per state and then looping over that state's counties avoids re-reading the same polygons again and again. If every county in a state (within this part) already has a checkpoint, the state is skipped before even attempting the CSB_GDB read, avoiding that cost entirely on a resumed run. Each county within the state is still individually checkpointed via `csb_county_path(geoid)`, with the same skip logic as the MMU model.

```r
source("/softs/R/createCluster.R")
cl <- createCluster()
clusterExport(cl, c(... "process_county_mmu", "tasks", "baseline_all", "MAKE_PLOTS"))
invisible(parLapplyLB(cl, seq_len(nrow(tasks)), function(i) {
  library(tidyverse); library(terra)
  process_county_mmu(geoid = tasks$GEOID[i], statefp = tasks$STATEFP[i], ...)
}))
stopCluster(cl)
### then a second, separate cluster cycle for CSB:
tasks_by_state <- split(tasks$GEOID, tasks$STATEFP)
cl <- createCluster()
clusterExport(cl, c(... "process_state_csb", "tasks_by_state", ...))
invisible(parLapplyLB(cl, names(tasks_by_state), function(statefp) {
  library(tidyverse); library(terra); library(sf)
  process_state_csb(statefp = statefp, geoids_in_state = tasks_by_state[[statefp]], ...)
}))
stopCluster(cl)
```
Two separate cluster. First the full MMU pass across every county in this part (`parLapplyLB` load-balances by county), then the CSB pass across every state represented in this part (`parLapplyLB` load-balances by state). They're kept separate because the two models parallelize along different units and have very different per-unit costs.

## Combining checkpoints into the final tables

```r
read_checkpoints <- function(dir) {
  files <- list.files(dir, pattern = "\\.rds$", full.names = TRUE)
  if (length(files) == 0) return(list(results = tibble(), quality = tibble()))
  all_data <- lapply(files, readRDS)
  list(results = bind_rows(lapply(all_data, `[[`, "results")),
       quality = bind_rows(lapply(all_data, `[[`, "quality")))
}

mmu_combined <- read_checkpoints(MMU_COUNTY_DIR)
csb_combined <- read_checkpoints(CSB_COUNTY_DIR)
all_results <- bind_rows(mmu_combined$results, csb_combined$results)
all_quality <- bind_rows(mmu_combined$quality, csb_combined$quality)

models_processing_details <- all_results %>% left_join(all_quality, by = c("GEOID", "model"))
```
`read_checkpoints()` simply finds every `.rds` file currently on disk for this part and row-binds them. `models_processing_details` is every model's per-category results joined with its confidence-quality metrics.

```r
models_processing_summary <- models_processing_details %>%
  group_by(GEOID, model) %>%
  summarise(
    bias_base      = sum(abs(delta_acres_baseline), na.rm = TRUE),
    bias_corrected = sum(abs(delta_acres_model), na.rm = TRUE),
    imp_share_pct  = 100 * sum(improvement_acres, na.rm = TRUE) / sum(abs(delta_acres_baseline), na.rm = TRUE),
    quality_ratio  = first(quality_ratio),
    .groups = "drop"
  )
```
While `models_processing_details` is saved at category level, `models_processing_summary` aggregate up to one row per `(GEOID, model)`. `bias_base` and `bias_corrected` sum absolute bias across all three categories (before/after correction), and `imp_share_pct` expresses the total improvement as a percentage of the total baseline bias. `quality_ratio` is taken via `first()` since it's already computed at the whole-raster level (one value per model per county, not per category) and is simply repeated across that model's category rows.

```r
missing_mmu <- setdiff(tasks$GEOID, sub("\\.rds$", "", basename(list.files(MMU_COUNTY_DIR, pattern = "\\.rds$"))))
missing_csb <- setdiff(tasks$GEOID, sub("\\.rds$", "", basename(list.files(CSB_COUNTY_DIR, pattern = "\\.rds$"))))
if (length(missing_mmu) == 0 && length(missing_csb) == 0) {
  cat("\nALL COUNTIES COMPLETE for part", WHICHPART, "-- no need to rerun.\n")
} else {
  cat("\nINCOMPLETE -- rerun this same script (WHICHPART =", WHICHPART, ") to retry the counties below...\n")
}
```
Compares this part's assigned task list against what's actually on disk after both passes, and prints exactly which GEOIDs are still missing from either model. Therefore, we can tell whether we need to rerun the script. Note that even if some GEOID are missing, we don't necessarily have to re-run as we will see in the later script (11_checkSkipped.R), it can happen for EXTREMELLY small counties that are therefore irrelevant for our analysis (and the models will always fail on them anyway), so we can just keep the baseline. Selecting the single best model per county (across all 18 MMU combinations and CSB) happens in a later script, once every part's outputs have been combined.
