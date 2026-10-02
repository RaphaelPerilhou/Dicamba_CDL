# `03_confidence_nat.R`

## Inputs
- `data/confidence_NAT_tiles/*.tif`: the GeoTIFF tiles (that Earth Engine automatically splits) are the national CDL confidence-band export. Produced by `export_CDL_confidence.js`  (see its readme (`readme_GEE_export_confidence.md`) for full details. They are manually downloaded from the `National_CDL_Confidence` Google Drive folder into the local directory before running this script.
- `data/2012_30m_cdls/2012_30m_cdls.tif` the raw, national-extent CDL cropland raster for 2012, used here only as a reference grid to align the merged confidence raster against.

## Outputs
- `data/confidence_NAT/CDL_conf_2012_national_merged.tif` (an intermediate output): all confidence tiles merged into a single national-extent raster, before alignment correction.
- `data/confidence_NAT/CDL_conf_2012_national_aligned.tif` (the final output): the merged confidence raster, cropped to exactly match the extent of the national CDL cropland raster, guaranteeing pixel-for-pixel alignment between the two.

## Summary
This script merges the separately-downloaded confidence-band tiles into a single national raster with `terra::vrt()`, verifies that this merged raster shares the same projection, resolution, and pixel origin as the official national CDL cropland raster, and then trims it to exactly the same extent.Currently implemented for 2012 only (`TARGET_YEAR <- 2012`) but can be extended by changing the year in this script as well as the GEE's one.

## Detailed explanation

```r
tiles <- list.files("data/confidence_NAT_tiles/", pattern = "\\.tif$", full.names = TRUE)
stopifnot(length(tiles) > 0)
conf_nat <- vrt(tiles)

writeRaster(conf_nat, "data/confidence_NAT/CDL_conf_2012_national_merged.tif", 
            overwrite = TRUE, datatype = "INT1U")
```
`terra::vrt()` builds a virtual raster (lighter than physically copying pixel data into memory), and joins all confidence tiles into one national-extent raster. `stopifnot(length(tiles) > 0)` is a guard against silently proceeding with an empty merge if the tiles directory is empty or misnamed. The virtual raster is then materialized with `writeRaster()` as `CDL_conf_2012_national_merged.tif`.

```r
conf_nat <- rast("data/confidence_NAT/CDL_conf_2012_national_merged.tif")
cdl_nat <- rast("data/2012_30m_cdls/2012_30m_cdls.tif")

stopifnot(crs(conf_nat) == crs(cdl_nat))
stopifnot(all(res(conf_nat) == res(cdl_nat)))
stopifnot(all(origin(conf_nat) == origin(cdl_nat)))
```
Re-reads the merged raster from disk `conf_nat` and loads the national CDL cropland raster as the reference grid. Three `stopifnot()` checks verify the two rasters share an identical coordinate reference system (`crs()`), identical pixel size (`res()`), and identical pixel grid origin (`origin()`).

```r
conf_aligned <- crop(conf_nat, ext(cdl_nat))

if (!compareGeom(conf_aligned, cdl_nat, stopOnError = FALSE)) {
  stop("Confidence raster does not align with national CDL after crop.")
}

writeRaster(conf_aligned,
            paste0("data/confidence_NAT/CDL_conf_", TARGET_YEAR,
                   "_national_aligned.tif"),
            overwrite = TRUE, datatype = "INT1U")
```
`crop(conf_nat, ext(cdl_nat))` trims the merged confidence raster down to exactly the bounding extent of the national CDL raster. Beczause the earlier checks already guaranteed matching CRS/resolution/origin, this crop is just removing any confidence pixels outside the CDL raster's extent, which are the ones we were not using anyway. `compareGeom(conf_aligned, cdl_nat, stopOnError = FALSE)` then does a final, explicit verification that the cropped raster now matches the CDL raster's geometry exactly (extent included this time). Once that check passes, the final aligned raster is written to `CDL_conf_2012_national_aligned.tif`. This is the file downstream scripts should read whenever they need a national confidence raster.
