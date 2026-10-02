# `00_setup_and_clip.R`

## Inputs
- `data/SF/states_2016.rds`: `sf` object of US state boundaries (produced by `000_get_sf_boundaries.R`). Used here only to derive the list of contiguous-US state FIPS codes.
- `data/SF/counties_2016.rds`: `sf` object of US county boundaries (produced by `000_get_sf_boundaries.R`). Provides `GEOID`, `STATEFP`, `COUNTYFP`, `NAME`, `LSAD`, and county polygon geometry, used both to build the county lookup table and to clip the national CDL raster to each county.
- `data/<year>_30m_cdls/<year>_30m_cdls.tif` for each year in `2009:2020`. These are the raw, national-extent USDA Cropland Data Layer (CDL) raster for its respective year, at 30m resolution, downloaded separately. Each pixel holds a numeric CDL land-cover code for that 30m cell, in the CDL's native Albers Equal Area projection.
- `/softs/R/createCluster.R`: This a cluster-setup script specific to the ANUBIS HPC cluster of TSE, parallelisation might need different setup if running on an other cluster. It provides `createCluster()` used for parallelizing the clipping step across worker nodes.

## Outputs
- `data/county_lookup.csv`: the reference table (one row per county in the 48 contiguous states) with columns `GEOID, STATEFP, COUNTYFP, NAME, LSAD`. Used throughout the rest of the project as the GEOID to county name lookup.
- `data/clipped/<STATEFP>/CDL_<year>_<GEOID>.tif`: one single-band raster per county per year (2009–2020), for every county in the 48 contiguous states. This is the national CDL raster cropped and masked to that county's boundary. Each raster has the same CDL numeric codes and 30m resolution as the source, but with all pixels outside the county polygon set to `NA`.

## Summary
This is the setup and preprocessing script. It has four jobs: (1) build a county name/ID lookup table used throughout the rest of the pipeline; (2) create the full output directory tree the later scripts expect to write into; (3) verify that all required national CDL raster files are present on disk and share a consistent projection, resolution, and extent across years; and (4) clip the national CDL raster for each year down to each individual county's boundary, in parallel on the ANUBIS HPC cluster, producing the per-county, per-year rasters that every subsequent script in the pipeline actually operates on (rather than repeatedly cropping the full national raster).

## Detailed explanation

```r
contiguous_statefps <- states_sf %>%
  sf::st_drop_geometry() %>%
  filter(!STATEFP %in% c("02", "15", "60", "66", "69", "72", "78")) %>%
  pull(STATEFP)
```
`sf::st_drop_geometry()` strips the geometry column so the object behaves like a plain data frame for this filtering step (geometry isn't needed to pick FIPS codes). The excluded codes are Alaska (`02`), Hawaii (`15`), and the outlying territories (American Samoa `60`, Guam `66`, the Northern Mariana Islands `69`, Puerto Rico `72`, and the US Virgin Islands `78`), leaving the 48 contiguous states plus DC. `TARGET_STATEFPS` and `TARGET_YEARS` (`2009:2020`) together define the full scope of counties × years the rest of the script (and pipeline) processes.

```r
counties_sf %>%
  sf::st_drop_geometry() %>%
  filter(STATEFP %in% TARGET_STATEFPS) %>%
  select(GEOID, STATEFP, COUNTYFP, NAME, LSAD) %>%
  write.csv("data/county_lookup.csv", row.names = FALSE)
```
Writes a plain CSV lookup table restricted to contiguous-US counties, keeping `GEOID` as the unique join key plus the human-readable `NAME` and the `LSAD` code (which, as noted in `000_get_sf_boundaries.R`'s README, flags independent cities via `LSAD == "25"`). This lets downstream scripts and figures label counties by name without having to carry the full `sf` object around.

```r
created <- 0
for (statefp in TARGET_STATEFPS){
  dirs <- file.path("data/clipped", statefp)
  for (d in dirs){
    if (!dir.exists(d)){
      dir.create(d, recursive = TRUE)
      created <- created + 1
    }
  }
}
```
Pre-creates the outputs' folders. The `if (!dir.exists(d))` ensures re-running the script won't touch folders that already exist. It also means the script won't pick up structural changes (e.g., a renamed subfolder) unless the old folders are deleted first.

```r
get_cdl_path <- function(year) {
  paste0("data/", year, "_30m_cdls/", year, "_30m_cdls.tif")
}

missing_files <- c()
for (year in TARGET_YEARS) {
  path <- get_cdl_path(year)
  if (file.exists(path)) { ... } else {
    missing_files <- c(missing_files, path)
  }
}
if (length(missing_files) > 0) {
  stop("Missing CDL files. ...")
}
```
`get_cdl_path()` centralizes the expected file-naming convention for raw CDL downloads (`data/<year>_30m_cdls/<year>_30m_cdls.tif`), so every other reference to a year's CDL raster in this script goes through the same function. Before doing any (potentially expensive, cluster-parallel) work, the script fails fast with `stop()` and a explicit list of missing paths if any year's file isn't present, rather than discovering the problem partway through the clipping loop.

```r
rasters <- lapply(TARGET_YEARS, function(y) rast(get_cdl_path(y)))
cat("CRS:       ", crs(rasters[[1]], describe = TRUE)$name, "\n")
cat("Resolution:", res(rasters[[1]]), "metres\n\n")

for (i in seq_along(rasters)[-1]) {
  cat("vs", TARGET_YEARS[i],
      "— CRS:", same.crs(rasters[[1]], rasters[[i]]),
      "| res:", all(res(rasters[[1]]) == res(rasters[[i]])),
      "| ext:", ext(rasters[[1]]) == ext(rasters[[i]]), "\n")
}
```
Loads all 12 years' national rasters (as lightweight `terra::rast()` pointers, not fully into memory) and checks the first year against every other year on three dimensions: `same.crs()` (identical coordinate reference system), matching `res()` (pixel size in the CDL's projected units = meters), and matching `ext()` (identical raster extent/bounding box). This is a sanity check that USDA hasn't changed the CDL's projection, resolution, or national extent across the years being pooled. Hence, if any of these silently differed across years, later steps that assume pixel-for-pixel alignment across years (e.g., building transition matrices) would be wrong without necessarily leading to an error.

```r
clipped_path <- function(year, geoid, statefp) {
  file.path("data/clipped", statefp, paste0("CDL_", year, "_", geoid, ".tif"))
}

clip_county <- function(year, geoid, statefp, counties_sf) {
  out_path <- clipped_path(year, geoid, statefp)
  if (file.exists(out_path)) { return(out_path) }  # skip if already done

  cdl         <- rast(get_cdl_path(year))
  county_vect <- counties_sf %>% filter(GEOID == geoid) %>% vect()
  county_proj <- project(county_vect, crs(cdl))
  clipped     <- crop(cdl, county_proj) %>% mask(county_proj, touches = FALSE)

  writeRaster(clipped, out_path, overwrite = TRUE, datatype = "INT1U")
  return(out_path)
}
```
`clip_county()` is the core per-county-per-year operation. It first checks whether the output already exists and skips if so (hence the whole clipping stage is resumable if any error on the cluster). Otherwise: `counties_sf %>% filter(GEOID == geoid) %>% vect()` extracts that single county's polygon and converts it from `sf` to a `terra::SpatVector`; `project()` reprojects the county polygon into the CDL raster's own CRS (rather than reprojecting the raster, which would be far more expensive and would resample pixel values); `crop()` then trims the raster to the county's bounding box, and `mask(..., touches = FALSE)` sets to `NA` every pixel whose *center* falls outside the county polygon (excluding pixels the county boundary merely touches/clips at the edge, rather than any pixel it overlaps at all).

```r
tasks <- counties_sf %>%
  sf::st_drop_geometry() %>%
  filter(STATEFP %in% TARGET_STATEFPS) %>%
  select(GEOID, STATEFP)

source("/softs/R/createCluster.R")
cl <- createCluster()

clusterExport(cl, c("counties_sf", "TARGET_YEARS",
                    "clip_county", "clipped_path", "get_cdl_path", "tasks"))

parLapplyLB(cl, seq_len(nrow(tasks)), function(i) {
  library(terra)
  library(dplyr)
  geoid   <- tasks$GEOID[i]
  statefp <- tasks$STATEFP[i]
  for (year in TARGET_YEARS) {
    clip_county(year, geoid, statefp, counties_sf)
  }
})

stopCluster(cl)
```
`tasks` enumerates one row per contiguous-US county (the parallelization unit is the county, with all 12 years for that county handled by a single worker). `createCluster()` launch a `parallel` cluster of worker processes. `clusterExport()` copies the objects and functions each worker needs into its own environment, since worker processes don't share memory with the main session. `parLapplyLB()` ("load-balanced" `parLapply`) dispatches county indices to workers dynamically as each finishes its previous task, rather than statically pre-splitting the work evenly. We use this rather than standard parLapply because different counties vary enormously in area (and thus clipping cost), so static splitting would leave some workers idle while others are still processing large counties. Each worker re-loads `terra` and `dplyr` (required since these aren't inherited from the main process) and loops over all `TARGET_YEARS` for its assigned county, calling `clip_county()` for each. `stopCluster(cl)` shuts down the worker processes once all tasks complete.
