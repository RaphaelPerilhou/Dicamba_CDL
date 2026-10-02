# `01bis_classification_dataframes.R`

## Inputs
- `outputs/<VERSION>/classification_summary_disagg_parts/<GEOID>.csv`: one CSV per county, produced upstream, holding raw per-CDL-code pixel counts for that county. Long format, one row per `(statefp, geoid, year, cdl_code)` combination, with a `n_pixels` column.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `"v1"` or `"v2"`, selected via `VERSION`): the CDL-code-to-category mapping metadata (documented in its own readme). Columns used here: `Keep`, `CDL Code`, `Category Code`.
- `data/SF/states_2016.rds` / `data/SF/counties_2016.rds`: `sf` boundary objects (produced by `000_get_sf_boundaries.R`), used here only to derive the list of contiguous-US state FIPS codes (`TARGET_STATEFPS`) and the study years (`TARGET_YEARS`).

## Outputs
- `outputs/<VERSION>/classification_summary_disagg.csv`, long format: every part-file simply concatenated into one CSV, one row per `(statefp, geoid, year, cdl_code)`, with `n_pixels`.
- `outputs/<VERSION>/classification_summary_disagg_wide.csv`, wide format of the same data: one row per `(statefp, geoid, year)`, one column per raw CDL code, values = pixel counts (0 where a code never occurs in that county-year).
- `outputs/<VERSION>/classification_summary.csv`: the aggregated category-level summary: one row per `(statefp, geoid, year)`, with columns `NonCrop, GM, Tolerant, Vulnerable, Unclassified, total` (pixel counts), recovered directly from the disaggregated long table via the CDL-code → category mapping, without re-touching any raster.


## Summary
This script takes the per-county disaggregated classification counts (raw pixel counts by CDL code, split across one file per county from an earlier parallel step) and consolidates them into three CSVs: a single long-format file, a wide pivot by CDL code, and a wide pivot by category (NonCrop/GM/Tolerant/Vulnerable/Unclassified). The category-level aggregation is done by mapping each raw CDL code to its category and summing pixel counts, rather than by re-running any raster reclassification, since the per-code pixel counts already contain everything needed to recover the per-category counts.

## Detailed explanation

```r
part_files <- list.files(SUMMARY_DISAG_PARTS_PATH, full.names = TRUE)
long <- read_csv(part_files, show_col_types = F)
write_csv(long, SUMMARY_DISAG_PATH)
```
`read_csv()` (from `readr`) accepts a vector of file paths and reads + row-binds them all into a single data frame in one call. Hence, this step merges the per-county part-files back into one national long-format table. That combined table is immediately written out as `classification_summary_disagg.csv`.

```r
wide <- long %>% 
  pivot_wider(
    id_cols = c(statefp, geoid, year),
    names_from = cdl_code,
    values_from = n_pixels,
    values_fill = 0
    )
write_csv(wide, WIDE_SUMMARY_DISAG_PATH)
```
`pivot_wider()` reshapes the long table so that each unique `cdl_code` value becomes its own column, with `n_pixels` as the cell values. `id_cols = c(statefp, geoid, year)` fixes what defines a row (one row per county-year), and `values_fill = 0` fills in `0` for any CDL code that simply never appears in a given county-year (rather than leaving `NA`, which would be ambiguous between "zero pixels" and "not recorded"). The result is one row per county-year with one column per raw CDL code.

```r
cdl_map <- read_excel(METADATA_XLSX, sheet = VERSION) %>%
  filter(Keep == 1)

code_to_category <- setNames(cdl_map$`Category Code`, as.character(cdl_map$`CDL Code`))
```
Reads the CDL-code-to-category mapping directly from the metadata workbook's `v1`/`v2` sheet (matching `VERSION`), keeping only rows with `Keep == 1` (i.e., CDL codes actually included in the analysis). `setNames()` turns this into a **named lookup vector**: names are CDL codes, values are the corresponding `Category Code` (0/1/2/3). This lets any CDL code be mapped to its category with a single vectorized indexing operation (`code_to_category[as.character(some_code)]`) rather than a join or a loop.

```r
category_labels <- c("0" = "NonCrop", "1" = "GM", "2" = "Tolerant",
                     "3" = "Vulnerable", "99" = "Unclassified")

aggregated <- long %>%
  mutate(
    category = code_to_category[as.character(cdl_code)],
    category = ifelse(is.na(category), 99, category)  # unmapped codes -> Unclassified
  ) %>%
  group_by(statefp, geoid, year, category) %>%
  summarise(n_pixels = sum(n_pixels), .groups = "drop")
```
For every row in the long table, `code_to_category[as.character(cdl_code)]` looks up that CDL code's category. Any CDL code with `Keep == 0` (and therefore filtered out when `cdl_map` was built) returns `NA`, and `ifelse(is.na(category), 99, category)` reassigns those to category `99` (Unclassified). `group_by(..., category) %>% summarise(n_pixels = sum(n_pixels))` then collapses all CDL codes sharing a category into a single pixel count per `(statefp, geoid, year, category)`. Thus, this recovers category-level totals from the disaggregated counts, with no raster re-processing.

```r
aggregated_wide <- aggregated %>%
  mutate(category_label = category_labels[as.character(category)]) %>%
  select(-category) %>%
  pivot_wider(
    id_cols = c(statefp, geoid, year),
    names_from = category_label,
    values_from = n_pixels,
    values_fill = 0
  ) %>%
  mutate(total = NonCrop + GM + Tolerant + Vulnerable + Unclassified) %>%
  select(statefp, geoid, year, NonCrop, GM, Tolerant, Vulnerable, Unclassified, total)

write_csv(aggregated_wide, AGGREGATED_WIDE_PATH)
```
Converts the numeric `category` codes to their text (GM, Tolerant...) labels via `category_labels`, then pivots wider the same way as before, this time by category label instead of raw CDL code. Hence, it produces one column per category (`NonCrop, GM, Tolerant, Vulnerable, Unclassified`). A `total` column is added as the row-wise sum of all five category columns, giving the total classified pixel count for that county-year, and the columns are reordered. `classification_summary.csv` is the file most downstream analysis scripts read, since it gives category-level pixel counts per county-year directly, without needing to re-derive them from the disaggregated data.
