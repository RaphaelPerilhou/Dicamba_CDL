# `08bis_confidence_baseline_disagg.R`

> Same idea as `08_confidence_baseline.R` (mean CDL confidence at baseline, to serve as the denominator for later model-quality ratios), computed at the raw-CDL-code / commodity level instead of the category level. See `readme08.md` for the general logic. Only what's different is covered below.

## Inputs
- `outputs/<VERSION>/class_and_conf_disagg/<STATEFP>/Conf_stacked_disagg_<YYYY>_<GEOID>.tif`: per-county two-band raster (raw CDL code + confidence), from `04bis_clip_confidence_disagg.R`.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx`: to map CDL codes to Census commodities.
- `outputs/<VERSION>/baseline_bias_disagg_commodity.csv`: (from `07bis`), joined at the end (in place of `08`'s `baseline_bias.csv`).
- `data/metadata/<VERSION>/missing_geoid_census.csv`, `data/county_lookup.csv`: same as elsewhere.

## Outputs
- `outputs/<VERSION>/quality_baseline_disagg.csv`: one row per `(GEOID, cdl_code)`, with pixel count and mean confidence per raw CDL code, plus each county's overall mean confidence.
- `outputs/<VERSION>/quality_baseline_disagg_commodity.csv`: the above aggregated to `(GEOID, commodity)`.
- `outputs/<VERSION>/baseline_measures_disagg_commodity.csv`: `07bis`'s bias table joined with the commodity-level confidence measures.

## What's actually different from `08`

```r
df <- tibble(cdl_code = code_vals[is_valid], confidence = conf_vals[is_valid]) %>%
  group_by(cdl_code) %>%
  summarise(n_pixels = n(), mean_conf_cdl_code = mean(confidence, na.rm = TRUE), .groups = "drop") %>%
  mutate(GEOID = geoid, mean_conf_county = mean_conf_county)
```
Where `08` restricts to just the 3 cropland categories and computes one mean per category, this groups by whatever raw CDL codes are actually present in the county and computes one mean confidence per code.

```r
code_to_commodity <- read_excel(METADATA_XLSX, sheet = VERSION) %>%
  filter(`Has Census` == 1) %>%
  select(`CDL Code`, `Census Commodity`) %>% distinct()

quality_baseline_commodity <- quality_baseline_disagg %>%
  left_join(code_to_commodity, by = c("cdl_code" = "CDL Code")) %>%
  filter(!is.na(`Census Commodity`)) %>%
  group_by(GEOID, commodity = `Census Commodity`) %>%
  summarise(
    mean_conf_commodity = weighted.mean(mean_conf_cdl_code, w = n_pixels),
    n_pixels = sum(n_pixels), .groups = "drop"
  )
```
This step is not needed in `08`. Because a commodity can span several CDL codes (e.g. Corn Grain + Corn Silage both map to CORN), the per-code means are aggregated up to commodity level with a pixel-weighted mean (`weighted.mean(..., w = n_pixels)`), not a plain average, so a code contributing more pixels correctly counts for more of the commodity's overall confidence.
