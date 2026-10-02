# `11_checkSkipped.R`

> This script follows up on `11_post_processing_models.R` and check for counties that `process_county_mmu()`/`process_state_csb()` failed to produce a checkpoint `.rds` for.

## Inputs
- `outputs/v2/baseline_measures.csv`: county × category bias/confidence table (from `08_confidence_baseline.R`), used here only for its `acres_census` column.
- `data/metadata/v2/missing_geoid_census.csv`: the list of GEOIDs with no usable Census of Agriculture data at all (from `05bis_census_area.R`), i.e. counties expected to be thin or absent from the bias table regardless of the post-processing run.
- The skipped-GEOID lists themselves written manually as one character vector per `WHICHPART` chunk (`PP1` through `PP10`), transcribed from the incomplete-county summaries that `11_post_processing_models.R`'s `read_checkpoints()` step reports at the end of each node's run.

## Outputs
- `data/metadata/v2/reclass_skipped_counties.csv`: the full list of skipped GEOIDs across all 10 chunks, for future runs to check against or exclude.

## Summary
For the handful of GEOIDs in each chunk that never got a checkpoint written by `100_ReclassX.R`, this script looks up the skipped GEOIDs' Census cropland acreage and checks whether they're already known to be missing from Census altogether. If the skipped counties are consistently tiny (or already excluded from Census-based comparisons), the missing model output for them can be safely ignored as they will not matter for the Dicamba Analysis anyway.

## Detailed explanation

```r
baseline_all %>%
  filter(GEOID %in% c("11001","08111","48443","21195")) %>%
  group_by(GEOID) %>%
  summarise(total_baseline_acres = sum(acres_census, na.rm = TRUE))

intersect(c("11001","08111","48443","21195"), missing_geoids$GEOID)
```
This pattern is repeated once per `WHICHPART` chunk (PP1 through PP10), each time the GEOID vector is filled with the ones, skipped by `100_ReclassX.R`, respectively . The first call sums `acres_census` across all 3 categories for the chunk's skipped counties, allowing us to have an idea of each GEOID agricultural area. The second line, `intersect()`, flags any of those GEOIDs that are in the `missing_geoids` list. If the latter holds, those counties were irrelevant for our analysis anyway.
