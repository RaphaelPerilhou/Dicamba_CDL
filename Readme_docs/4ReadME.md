# `04_clip_confidence.R`

## Inputs
- `outputs/<VERSION>/classified/<STATEFP>/Classified_<YYYY>_<GEOID>.tif`: the per-county, per-year category raster (produced by `01ter_classification_rasters.R`), values `{0, 1, 2, 3, 99, NA}`.
- `data/confidence_NAT/CDL_conf_2012_national_aligned.tif`: the national CDL confidence-band raster for 2012, pre-aligned to the national CDL cropland grid (produced by `03_confidence_nat.R`, see also the GEE export readme for how this raster's tiles were generated).
- `data/SF/counties_2016.rds`: `sf` object of US county boundaries, used to derive per-county clipping polygons for the confidence raster.
- `data/county_lookup.csv`: the `GEOID`/`STATEFP` lookup table (produced by `00_setup_and_clip.R`), used here to enumerate every county to process.

## Outputs
- `outputs/<VERSION>/class_and_conf/<STATEFP>/Conf_stacked_<YYYY>_<GEOID>.tif`: a two-band raster per county (currently 2012 only), band 1 named `"category"` (the classified category raster, unchanged) and band 2 named `"confidence"` (that county's confidence values, clipped from the national confidence raster), guaranteed pixel-for-pixel aligned with band 1.

## Summary
This script attaches the CDL confidence band to each county's classified category raster, producing a single two-band file per county that carries both category and confidence per pixel. Rather than re-deriving confidence from scratch per county, it clips the already-aligned national confidence raster (from `03_confidence_nat.R`) down to each county's boundary, using the exact same clipping approach (`crop()` + `mask(touches = FALSE)` on a projected county polygon) as `00_setup_and_clip.R` uses to clip the national CDL raster. This guarantees the clipped confidence raster lines up pixel-for-pixel with the county's classified raster. Currently scoped to a single year (`TARGET_YEAR <- 2012`), as it's the year used later on post-processing, where confidence will be needed.

## Detailed explanation

```r
counties_proj <- project(vect(counties_sf), crs(conf_nat))
```
Converts the counties `sf` object to a `terra::SpatVector` and reprojects all counties at once into the confidence raster's CRS, done a single time before the loop rather than reprojecting one county polygon per iteration.

```r
for (i in seq_len(nrow(tasks))) {
  geoid   <- tasks$GEOID[i]
  statefp <- tasks$STATEFP[i]
  year    <- TARGET_YEAR

  out_path <- CONF_OUTPUT_PATH(year, geoid, statefp)
  if (file.exists(out_path)) next

  classified_file <- CLASSIFIED_PATH(year, geoid, statefp)
  if (!file.exists(classified_file)) next
  ...
```
The loop runs without parallization (lighter job but still take a fair amount of time depending on the machine). Each iteration is resumable (`if (file.exists(out_path)) next` skips counties already processed). 

```r
classified  <- rast(classified_file)
county_proj <- counties_proj[counties_proj$GEOID == geoid, ]
confidence  <- crop(conf_nat, county_proj) %>% mask(county_proj, touches = FALSE)

if (!compareGeom(classified, confidence, stopOnError = FALSE)) {
  warning("Extent mismatch GEOID ", geoid, " -- skipping.")
  next
}
```
`county_proj <- counties_proj[counties_proj$GEOID == geoid, ]` pulls out that one county's already-reprojected polygon from the pre-projected set. `crop(conf_nat, county_proj) %>% mask(county_proj, touches = FALSE)` clips the national confidence raster to that county (`crop()` trims to the polygon's bounding box), `mask(..., touches = FALSE)` then sets to `NA` every pixel whose center falls outside the actual county boundary (not just its bounding box), excluding edge-touching pixels the same way `00_setup_and_clip.R`'s `clip_county()` does for the CDL raster itself. `compareGeom(classified, confidence, stopOnError = FALSE)` verifies the operation does not disalign the rather.

```r
stacked        <- c(classified, confidence)
names(stacked) <- c("category", "confidence")
writeRaster(stacked, out_path, overwrite = TRUE, datatype = "INT1U")
```
`c(classified, confidence)` combines the two single-band `SpatRaster` objects into one two-band `SpatRaster` (this oworks cleanly because their geometries match, as we just verified above). `names(stacked) <- c("category", "confidence")` labels the two bands explicitly, so any script reading this file later can select `stacked$category` or `stacked$confidence` by name rather than by band index.
