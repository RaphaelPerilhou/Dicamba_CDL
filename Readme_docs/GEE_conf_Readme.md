# GEE script: `export_CDL_confidence.js` (Google Earth Engine)

## Inputs
- `USDA/NASS/CDL/2012`: the USDA/NASS Cropland Data Layer Earth Engine image collection for 2012. This script selects only its `confidence` band (a per-pixel classification confidence score, separate from the `cropland` land-cover band already obtained via the R pipeline's own CDL downloads).

## Outputs
- Google Drive folder `National_CDL_Confidence/CDL_conf_2012_national*.tif`: the full national-extent CDL confidence band for 2012, with no county clipping (`region` omitted, so Earth Engine defaults to the image's native/full extent). Because this covers the entire contiguous US at 30m resolution, Earth Engine's export cannot produce it as a single file and therefore splits the output into multiple GeoTIFF tiles in the same Drive folder. **IMPORTANT:** These are the tiles that must be downloaded into `data/confidence_NAT_tiles/` for `03_confidence_nat.R`.

## Summary
This script pulls the CDL confidence band (indicating per-pixel confidence in that year's classification) from the CDL and exports the full national raster, unclipped, to Google Drive. The key technical concern is grid alignment as a raster with a slightly different pixel grid would break any later pixel-for-pixel comparison against the R-side CDL rasters. Hence, the export forces the output onto CDL's own native grid explicitly.

## Detailed explanation

```javascript
var confidence = ee.Image("USDA/NASS/CDL/2012").select("confidence");
```
Loads the 2012 CDL image from Earth Engine's hosted CDL and keeps only the `confidence` band, discarding `cropland`.

```javascript
Export.image.toDrive({
  image:          confidence,
  description:    "CDL_conf_2012_national",
  folder:         "National_CDL_Confidence",
  crs:            "EPSG:5070",
  crsTransform:   [30, 0, -2356095, 0, -30, 3172605],
  maxPixels:      1e13,
  fileFormat:     "GeoTIFF"
  // pas de region = extent native du CDL
});
```
`crs: "EPSG:5070"` (NAD83 / Conus Albers) is CDL's native projection. `crsTransform: [30, 0, -2356095, 0, -30, 3172605]` is encoding CDL's exact 30m pixel size and native grid origin. Passing both `crs` and `crsTransform` explicitly ensures the exported raster uses the exact same pixel grid as the official CDL rasters.

No `region` argument is given, so the export defaults to the image's own full native extent (National coverage). `maxPixels: 1e13` raises Earth Engine's default per-export pixel threshold above its default, needed since a national 30m raster has an enormous number of pixels. Because this exceeds Google Drive's/Earth Engine's per-file export size limit, Earth Engine splits the result into several separate GeoTIFF tiles within the `National_CDL_Confidence` Drive folder. Every tile still shares the exact same CRS and `crsTransform` (and therefore the same underlying pixel grid), later allowing `terra::vrt()` on the R script to join them back into one raster.

### Downloading the output

Once the files are on the Google Drive, every resulting tiles must be downloaded from it and put into `data/confidence_NAT_tiles/` on the machine running `03_confidence_nat.R` .
