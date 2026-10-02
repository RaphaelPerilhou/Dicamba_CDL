# `12_0_Post_processing_cleaning.R`

## Inputs
- `outputs/v2/parts/part_<1..10>/mmu_by_county/*.rds`, `outputs/v2/parts/part_<1..10>/csb_by_county/*.rds`: per-county checkpoint files from `11_post_processing_models.R`'s 10 `WHICHPART` runs.
- `outputs/v2/parts/part_<1..10>/models_processing_details.csv`, `models_processing_summary.csv` each part's own processing-log CSVs, also written by `11`.

## Outputs
- `outputs/v2/post_processing/mmu_by_county/*.rds`, `outputs/v2/post_processing/csb_by_county/*.rds`: all parts' checkpoint files merged into two single folders.
- `outputs/v2/post_processing/models_processing_details.csv`, `models_processing_summary.csv`: each part's version of these two files row-bound into one combined file.

## Summary
Clean outputs of `11_post_processing_models.R` for next steps. Since `11` ran as 10 separate `WHICHPART` jobs, each outputs are initially on its own `part_<i>/` subfolder. this script merges all 10 parts into one `post_processing/` directory. We end up with a single national dataset.

## Detailed explanation

```r
for (subdir in c("mmu_by_county", "csb_by_county")) {
  out_dir <- file.path(POST_PROCESSING_DIR, subdir)
  dir.create(out_dir, recursive = TRUE, showWarnings = FALSE)
  for (i in 1:N_PARTS) {
    source_dir <- file.path(PARTS_DIR, paste0("part_", i), subdir)
    if (!dir.exists(source_dir)) next
    
    files <- list.files(source_dir, pattern = "\\.rds$", full.names = TRUE)
    file.copy(files, out_dir, overwrite = TRUE)
  }
}
```
For each of the two checkpoint types, loops over all 10 parts and copies every `.rds` file it finds into the single shared output folder. Since county GEOIDs are unique across parts, there's no risk of one part's checkpoint overwriting another's.

```r
for (subfile in c("models_processing_details.csv", "models_processing_summary.csv")) {
  all_rows <- list()
  for (i in 1:N_PARTS) {
    csv <- file.path(PARTS_DIR, paste0("part_", i), subfile)
    all_rows[[i]] <- read_csv(csv, show_col_types = FALSE)
  }
  combined <- bind_rows(all_rows)
  write_csv(combined, file.path(POST_PROCESSING_DIR, subfile))
}
```
Same idea. It reads each part's copy into a list and `bind_rows()`s them into one combined file, so the full national processing log (and summary) can be read as a single table rather than 10 separate per-part files.
