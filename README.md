# Dicamba LLULC project

This project builds, corrects, and validates a county-level land-cover classification derived from the USDA Cropland Data Layer (CDL), grouping crops into four Dicamba-relevant categories: GM (dicamba-tolerant, genetically modified), Tolerant, Vulnerable, and NonCrop (plus an Unclassified category). Then, it compares that classification against the USDA Census of Agriculture. It then applies two post-processing correction methods (MMU spatial filtering and CSB field-modal correction) to reduce CDL's bias relative to Census, selects the best-performing correction per county, and validates that correction out-of-sample against an independent Census year.

Every script has its own detailed readme under [`Readme_docs/`](Readme_docs/), written in the same format: **Inputs**, **Outputs**, **Summary**, and a **Detailed explanation** that walks through the actual code. This README ties them together and make its easier to navigates across each of them.

`CDL_CENSUS_MAP_meta.xlsx` is read directly by nearly every script and a sheet can be added to the file to use different categories, mask and `short_desc` for Census.

## 1. Key inputs and outputs

Some files are used multiple time across different scripts. Everything else is documented in its own script's readme. This section is for the things that should be understood beforehand.

### `data/metadata/CDL_CENSUS_MAP_meta.xlsx`
The single mapping excel file nearly every script reads from. It decides, per raw CDL code: whether to keep it in the analysis (`Keep`), which of the four categories it belongs to (`Category`/`Category Code`), and, if it has one, which Census of Agriculture commodity and `short_desc` line(s) (`Census Commodity`, `Has Census`, `Census Short_desc`) correspond to it. It has two sheets: **v1** (frozen historical record of the original mapping) and **v2** (current/authoritative sheet that drops the noisy Other Hay/Clover/Vetch codes and isolates Alfalfa from the aggregated HAY line). `VERSION` at the top of nearly every script simply picks which sheet to read. Full details: [`Readme_docs/CDL_CENSUS_metadata.md`](Readme_docs/CDL_CENSUS_metadata.md).

### `data/county_lookup.csv` and the `data/SF/*.rds` boundary files
`county_lookup.csv` (`GEOID`, `STATEFP`, `COUNTYFP`, `NAME`, `LSAD`) is the list of the ~3,000 contiguous-US counties, and the join key (`GEOID`) used throughout. It and the `sf` boundary objects it's derived from (`states_2016.rds`, `counties_2016.rds`) are built once, at the very start of the project (`000_get_sf_files.R`, `00_setup_and_clip.R`), and read by nearly every later script that needs to enumerate counties or clip/mask a raster to one.

### The raster families
Each of these exists as one file per county per year, under `outputs/<VERSION>/<family>/<STATEFP>/`:
- **`classified_disagg` (`Disagg_*.tif`)** : raw CDL codes, masked to the county's union agricultural footprint (a pixel counts if it was agricultural in any study year). This is the finest-grained raster and the source for everything downstream.
- **`classified` (`Classified_*.tif`)**: the same footprint, reclassified into the 5-category scheme (`0`=NonCrop, `1`=GM, `2`=Tolerant, `3`=Vulnerable, `99`=Unclassified).
- **`class_and_conf` / `class_and_conf_disagg`**: the classified/disaggregated raster stacked with the CDL confidence band (2012 only), used for the post-processing model search and its quality threshold.
- **`final_corrected` (`Final_*.tif`)**: the classified raster after each county's single best-performing post-processing correction (MMU, CSB, or no correction at all) has been applied. This is the project's end classification product and what every downstream analysis and the 2017 validation actually measures.

### Bias, quality, and model-selection tables
- **`baseline_bias.csv` / `baseline_measures.csv`**: county × category CDL-vs-Census acreage bias, plus CDL classification-confidence baselines. Every correction model is scored against this (from the 2012 selecting year).
- **`models_processing_summary.csv` and `best_per_county_q1.csv`**: the post-processing grid search's raw output, collapsed to a single winning model per county (subject to a `quality_ratio < 1` to mitigate the risk of  a model improving" bias by coincidence rather than by correcting genuinely low-confidence pixels. This file is what `13_apply_best_model.R` reads to know which correction (if any) to apply into the final rasters.
- **Transition matrices (`transition/TM_*.csv` baseline vs. `transition_final/TM_*.csv` corrected)**: year-over-year category pixel counts, compared before/after correction in `16_transition_analysis.R`.

### Constants and conventions used everywhere
- **`PIXEL_ACRES = 0.222395`**: the fixed pixel-to-acres conversion (one 30m CDL pixel in acres).
- **`VERSION` (`"v1"`/`"v2"`)**: selects which sheet of the metadata xlsx (and which downstream output subtree) a script uses; `v2` is the current/authoritative mapping.
- **ANUBIS cluster pattern**: every raster-heavy script parallelizes county-by-county (or state-by-state, for CSB) across the ANUBIS HPC cluster using `source("/softs/R/createCluster.R")` + `parLapplyLB()`, and is individually resumable via per-file/per-county checkpointing, so a partially-completed cluster run can always be safely re-submitted.

## 3. Script index
This gives a very short summary of each script and also respect the ordrer in which the scripts should be run for full replication.

| # | Script | What it does | Docs |
|---|--------|---------------|------|
| 0a | `000_get_sf_files.R` | Downloads and caches US state/county boundary shapefiles. | [Readme_docs](Readme_docs/000ReadMe.md) |
| 0b | `00_setup_and_clip.R` | Builds `county_lookup.csv`, and clips the national CDL raster to every county/year. | [Readme_docs](Readme_docs/00ReadMe.md) |
| 1a | `01_mask_disagg.R` | Builds each county's union agricultural mask and applies it to every year's raw-CDL raster. | [Readme_docs](Readme_docs/1ReadMe.md) |
| 1b | `01bis_classification_dataframes.R` | Consolidates per-county pixel counts into national long/wide, code-level and category-level CSVs. | [Readme_docs](Readme_docs/1bisReadMe.md) |
| 1c | `01ter_classification_rasters.R` | Reclassifies the disaggregated rasters into the 5-category classified rasters. | [Readme_docs](Readme_docs/1terReadMe.md) |
| 2 | `02_TM_disag.R` | Computes baseline (uncorrected) year-over-year transition matrices, code-level and category-level. | [Readme_docs](Readme_docs/2ReadMe.md) |
| — | `export_CDL_confidence.js` | Google Earth Engine script exporting the national CDL confidence band to Drive. | [Readme_docs](Readme_docs/GEE_conf_ReadMe.md) |
| 3 | `03_confidence_nat.R` | Merges the downloaded confidence tiles and aligns them to the national CDL grid. | [Readme_docs](Readme_docs/3ReadMe.md) |
| 4a | `04_clip_confidence.R` | Clips the national confidence raster to each county and stacks it onto the classified raster. | [Readme_docs](Readme_docs/4ReadMe.md) |
| 4b | `04bis_clip_confidence_disagg.R` | Same as `04`, stacked onto the disaggregated (raw-code) raster instead. | [Readme_docs](Readme_docs/4bisReadMe.md) |
| 5a | `05_Availability_Census.R` | Exploratory: checks Census county-level coverage and decides which `short_desc` lines to sum per commodity. | [Readme_docs](Readme_docs/5ReadMe.md) |
| 5b | `05bis_census_area.R` | Production: turns the metadata xlsx's mapping decisions into the actual filtered 2012 Census acreage table. | [Readme_docs](Readme_docs/5bisReadMe.md) |
| 5c | `5ter_Hay_exploration.R` | Exploratory: justifies isolating Alfalfa from the aggregated HAY commodity. | [Readme_docs](Readme_docs/5terReadMe.md) |
| 6 | `06_census_acres.R` | Aggregates Census acreage to commodity and category level, handling disclosure-suppressed values. | [Readme_docs](Readme_docs/6ReadMe.md) |
| 7a | `07_bias_measure.R` | Computes the core CDL-vs-Census bias/accuracy metrics per county × category. | [Readme_docs](Readme_docs/7ReadMe.md) |
| 7b | `07bis_bias_measure_disagg.R` | Same bias metrics, at county × commodity level. | [Readme_docs](Readme_docs/7bisReadMe.md) |
| 8a | `08_confidence_baseline.R` | Computes baseline mean CDL confidence per county/category, for later quality-ratio comparisons. | [Readme_docs](Readme_docs/8ReadMe.md) |
| 8b | `08bis_confidence_baseline_disagg.R` | Same, at raw-CDL-code/commodity granularity. | [Readme_docs](Readme_docs/8bisReadMe.md) |
| 10 | `100_Reclass_X.R` (×11 parts) | Grid-searches 18 MMU parameter combinations plus the CSB field-modal model per county, scored against Census. | [Readme_docs](Readme_docs/10ReadMe.md) |
| 11 | `11_checkSkipped.R` | Checks that counties skipped by the post-processing search are negligibly small / already excluded from Census. | [Readme_docs](Readme_docs/11ReadMe.md) |
| 12a | `12_0_Post_processing_cleaning.R` | Merges the 10 parallel parts' post-processing outputs into one national dataset. | [Readme_docs](Readme_docs/12_0ReadMe.md) |
| 12b | `12_post_processing_analysis.R` | Selects each county's single best correction model and produces the improvement figures/tables. | [Readme_docs](Readme_docs/12ReadMe.md) |
| 13 | `13_apply_best_model.R` | Applies each county's winning model across all 12 study years to produce the final corrected rasters. | [Readme_docs](Readme_docs/13ReadMe.md) |
| 14 | `14_classification_summary_final.R` | Re-runs the pixel-count summary (`01bis`'s counterpart) on the final corrected rasters. | [Readme_docs](Readme_docs/14ReadMe.md) |
| 15 | `15_Transition_final.R` | Computes year-over-year transition matrices on the corrected rasters. | [Readme_docs](Readme_docs/15ReadMe.md) |
| 16 | `16_transition_analysis.R` | Compares baseline vs. corrected transition matrices (3×3 GM/Tolerant/Vulnerable), pooled at 3 levels. | [Readme_docs](Readme_docs/16ReadMe.md) |
| 17 | `17_Census_validation_year.R` | Validates each county's fixed, 2012-selected model against independent 2017 Census data. | [Readme_docs](Readme_docs/17ReadMe.md) |

Figures and tables produced along the way carry their own interpretation in the dissertation report.
