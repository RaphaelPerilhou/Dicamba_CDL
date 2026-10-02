# `000_get_sf_files.R`

## Inputs
- None (pulls directly from the US Census Bureau via the `tigris` package's API/FTP access).

## Outputs
- `data/SF/states_2016.rds` — an `sf` object of US state boundaries (2016 vintage, cartographic boundary/`cb = TRUE` resolution). Each row is one state; geometry column holds the polygon.
- `data/SF/counties_2016.rds` — an `sf` object of US county boundaries (2016 vintage, `cb = TRUE`). Each row is one county (or county-equivalent, e.g. independent cities), with a `GEOID` column (5-digit state+county FIPS code) and a `NAME` column (raw Census name), plus a geometry column.

## Summary
This script downloads US state and county boundary shapefiles once, using the `tigris` package, and caches them locally as `.rds` files. It exists specifically because the project's HPC cluster (ANUBIS) does not have `tigris` installed and/or reliable internet access to hit the Census TIGER/Line API — so boundaries are fetched here, on a machine that does have `tigris` and internet access, and saved as portable `sf` objects that can be loaded anywhere downstream (including on ANUBIS) with a plain `readRDS()`, no package or network dependency required.

## Detailed explanation

```r
states_sf <- states(year = 2016, cb = TRUE)
counties_sf <- counties(year = 2016, cb = TRUE)
```
`tigris::states()` and `tigris::counties()` each download a Census cartographic boundary ("cb") shapefile for the requested `year` and return it as an `sf` object. `cb = TRUE` requests the generalized (simplified) cartographic boundary files rather than the full-resolution TIGER/Line files — smaller, faster to load, and appropriate here since these boundaries are used for spatial joins/aggregation rather than precise line-work. `year = 2016` pins the vintage to match the CDL/Census years used elsewhere in the pipeline, since county boundaries and FIPS codes can change over time (splits, mergers, renumbering) and using a consistent, fixed vintage avoids `GEOID` mismatches against other years' data.

A comment in the script notes that `LSAD == "25"` in the counties shapefile identifies independent cities (per the Census Bureau's [legal/statistical area description codes](https://www.census.gov/library/reference/code-lists/legal-status-codes.html)) — i.e., places that function as county-equivalents without being a "county" in name. No filtering or relabeling is done here; it's left as a note for downstream scripts, since `GEOID` (not `NAME`) is the key used throughout the rest of the pipeline, so independent cities don't need special handling to be correctly identified.

```r
dir.create("data/SF/", showWarnings = F, recursive = T)
saveRDS(states_sf, "data/SF/states_2016.rds")
saveRDS(counties_sf, "data/SF/counties_2016.rds")
```
`dir.create(..., recursive = TRUE)` creates the `data/SF/` output folder if it doesn't already exist (and does so silently, `showWarnings = FALSE`, if it does). Both `sf` objects are then serialized with `saveRDS()` rather than exported as shapefiles/GeoPackages — since they're consumed exclusively by other R scripts in this pipeline, `.rds` preserves the exact `sf`/`data.frame` structure (including the CRS and geometry column type) without any format-conversion loss, and loads back with a single `readRDS()` call with no dependency on `tigris`, `sf`'s file drivers being available, or network access.
