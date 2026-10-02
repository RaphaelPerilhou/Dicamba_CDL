# `13_apply_best_model.R`

> `apply_mmu()`/`apply_csb()` here follow the same MMU/CSB logic as `run_mmu_model()`/`run_csb_model()` in `10_Reclass_X.R` (see `10ReadMe.md` for how the MMU patch-and-refill and CSB field-modal methods actually work). What's new in this script is covered below. Instead of grid-searching many candidate models per county for a single evaluation year (2012), it applies each county's single winning model (from `12_post_processing_analysis.R`'s `best_per_county`) across every year 2009–2020, in order to produce the final corrected rasters.

## Inputs
- `outputs/<VERSION>/classified/<STATEFP>/Classified_<YYYY>_<GEOID>.tif`: per-county-year classified rasters.
- `outputs/<VERSION>/post_processing/best_per_county_q1.csv`: one row per `GEOID` with its winning `best_model` name (e.g. `"MMU_5ac_nb8_w3_iter3"`, `"CSB_field_modal"`, or `"Baseline"`), from the model-selection step in `12_post_processing_analysis.R`.
- `data/county_lookup.csv`, `data/SF/counties_2016.rds`: county metadata and boundary polygons, as elsewhere.
- `data/metadata/<VERSION>/reclass_skipped_counties.csv`: the counties `10.R` never produced checkpoints for (see `11ReadMe.md`) are excluded here too, since they have no `best_model` assignment to apply.
- `data/NationalCSB_2013-2020_rev23/CSB1320.gdb`: CSB field boundaries, needed only for counties whose winning model is CSB.

## Outputs
- `outputs/<VERSION>/final_corrected/<STATEFP>/Final_<YYYY>_<GEOID>.tif`: one raster per county-year, 2009–2020: the classified raster with each county's single winning correction model (can be baseline) applied.

## Summary
This is the script that produces the final classification rasters. If takes each county's best model and applies it to all 12 years of classified rasters.

## Detailed explanation

```r
best_assignment <- read_csv(BEST_MODEL_ASSIGNMENT_PATH, show_col_types = FALSE) %>%
  mutate(
    GEOID = sprintf("%05d", as.integer(GEOID)),
    model_type = case_when(
      best_model == "Baseline" ~ "Baseline",
      best_model == "CSB_field_modal" ~ "CSB",
      str_detect(best_model, "^MMU_") ~ "MMU",
      TRUE ~ NA_character_
    ),
    mmu_acres      = if_else(model_type == "MMU",
                             as.numeric(str_extract(best_model, "(?<=MMU_)[0-9.]+(?=ac)")), NA_real_),
    mmu_neighbors  = if_else(model_type == "MMU",
                             as.integer(str_extract(best_model, "(?<=nb)[0-9]+")), NA_integer_),
    ...
```
Since `best_per_county_q1.csv` only stores the winning model as a name string (e.g. `"MMU_5ac_nb8_w3_iter3"`), this helps recovering the actual parameters (`mmu_acres`, `mmu_neighbors`, `mmu_window`, `mmu_iter`) needed to re-call `apply_mmu()` by pulling them back out of the string (`(?<=MMU_)[0-9.]+(?=ac)` grabs the number sitting between `MMU_` and `ac`, etc.). `model_type` collapses every county into one of three buckets (`Baseline`, `MMU`, `CSB`), which is what determines which of the two parallel passes below a county's tasks fall into.

```r
county_lookup <- county_lookup %>%
  ...
  left_join(best_assignment %>% select(...), by = "GEOID")
county_lookup <- county_lookup %>% anti_join(skipped_exclusion, by = "GEOID")
```
Excluding the already-known skipped counties (`anti_join` against `reclass_skipped_counties.csv`).

```r
baseline_mmu_tasks <- county_lookup %>%
  filter(model_type %in% c("Baseline", "MMU")) %>%
  tidyr::crossing(year = TARGET_YEARS)
```
Baseline and MMU counties are done together in one pass, parallelized per county-year task (`crossing()` expands each county into 12 rows, one per year). A "Baseline" county isn't actually run through any correction function at all: inside `process_baseline_or_mmu()`, `if (model_type == "Baseline") { writeRaster(classified, out_path, ...); return(...) }` just copies the already-classified raster straight through to `final_corrected/`, so every county ends up with a file in that directory regardless of whether a correction was applied.

```r
csb_counties <- county_lookup %>% filter(model_type == "CSB")
csb_states <- split(csb_counties$GEOID, csb_counties$STATEFP)
```
CSB counties are pulled into a second pass and grouped by state (`split()` gives a named list of GEOID vectors, one per `STATEFP`). `st_read(CSB_GDB, ...)` is the expensive step, so it's done once per state rather than once per county.

```r
remaining <- expand.grid(geoid = geoids_in_state, year = years, stringsAsFactors = FALSE)
remaining$done <- file.exists(FINAL_PATH(remaining$year, remaining$geoid, statefp))
remaining <- remaining[!remaining$done, ]
if (nrow(remaining) == 0) {
  cat("STATEFP", statefp, "-- all done, skipping.\n")
  return(invisible(NULL))
}
```
This script checkpoints per county-year file directly, such that before paying the cost of reading the state's CSB layer at all, it checks whether every county-year output this state needs already exists, and skips the `st_read()` call entirely if so.
