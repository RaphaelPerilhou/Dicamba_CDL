# `12_post_processing_analysis.R`

## Inputs
- `outputs/v2/post_processing/models_processing_summary.csv`: one row per `(GEOID, model)`, from `12_0_Post_processing_cleaning.R`'s merge of `11_post_processing_models.R`'s output: `bias_base` (uncorrected bias), `bias_corrected` (bias after applying that model), `quality_ratio`, `imp_share_pct` (improvement share).
- `outputs/v2/post_processing/models_processing_details.csv`: one row per `(GEOID, Category)`, supplying `acres_census`, used here only to build each county's total Census cropland acreage.

## Outputs
Figures (`figures/v2/post_processing/`): `p_bias_all.png`, `p_bias_by_bestmodel.png`, `p_best_imp.png`, `p_best_improvement.png`, `p_improvement_vs_severity.png`, `p_bias_by_severity_bin.png`, `p_bias_by_severity_bin_relative_trimmed.png`, `p_pct_improvement_by_bin.png`, `p_pct_improvement_by_bin_relative_trimmed.png`.

Tables (`tables/v2/post_processing/`): `table_best_model_per_county.tex`, `table_severity_summary_absolute.tex`, `table_severity_summary_relative.tex`.

- `outputs/v2/post_processing/best_per_county_q1.csv`, one row per `GEOID`, with `bias_base`, `best_model`, `best_bias`, `imp_share_pct`: This is giving the best performing model (including baseline) per-county, and it's used downstream by `13_apply_best_model.R`.

## Summary
This script uses `11_post_processing_models.R`'s per-county per-model output to produce our analysis of post-processing reclassification. For each county, it takes the single best-performing correction model, subject to the `quality_ratio < 1` to mitigate the risk of model improving accidentally the bias. Then, it compares that "best achievable" bias against the uncorrected baseline, and produces both the summary figures and the LaTeX tables reporting how much the MMU/CSB corrections improve on raw CDL.

## Detailed explanation

```r
baseline_df_all <- all_summary %>%
  distinct(GEOID, bias_base) %>%
  transmute(GEOID, model = "Baseline", bias = bias_base)

models_df_all <- all_summary %>%
  filter(quality_ratio < 1) %>%
  transmute(GEOID, model, bias = bias_corrected)
```
`quality_ratio < 1` keeps only `(GEOID, model)` pairs where the pixels that model reclassified were, on average, less confidently classified to begin with than the county's cropland as a whole. Any model-county combination that fails this test is simply excluded from `models_df_all`. `baseline_df_all` is the dataset keeping only the baseline result (uncorrected bias) for each county. Then, we stack together baseline and models results via `bind_rows()` to build the first boxplot (`p_bias_all.png`) comparing every model (with `quality_ratio <1`) against Baseline across all counties at once.

```r
label_model <- function(x) {
  dplyr::case_when(
    x == "Baseline" ~ "Baseline",
    x == "CSB_field_modal" ~ "CSB (modal)",
    grepl("^MMU_", x) ~ gsub(
      "^MMU_([0-9]+)ac_nb([0-9]+)_w([0-9]+)_iter[0-9]+$",
      "MMU (\\1ac, \\2nb, \\3w)",
      x
    ),
    TRUE ~ x
  )
}
```
This function is just used to get more readable names for our models on the outputs: goinf from something like `MMU_5ac_nb8_w3_iter3` to `MMU (5ac, 8nb, 3w)`.

### Choosing the best model per county

```r
best_per_county <- all_summary %>%
  distinct(GEOID, bias_base) %>%
  left_join(
    all_summary %>%
      filter(quality_ratio < 1) %>%
      group_by(GEOID) %>%
      slice_min(bias_corrected, n = 1, with_ties = FALSE) %>%
      ungroup() %>%
      select(GEOID, best_model_candidate = model, best_model_bias = bias_corrected, imp_share_pct),
    by = "GEOID"
  ) %>%
  mutate(
    is_better  = !is.na(best_model_bias) & best_model_bias < bias_base,
    best_model = if_else(is_better, best_model_candidate, "Baseline"),
    best_bias  = if_else(is_better, best_model_bias, bias_base)
  ) %>%
  select(GEOID, bias_base, best_model, best_bias, imp_share_pct)

write_csv(best_per_county, file.path(POST_PROCESSING_DIR, "best_per_county_q1.csv"))
```
This is to select the best model. We apply it to each county, separately, since counties are heterogeneous and therefore might be best served by different MMU parameter settings or by CSB. For each `GEOID`, `slice_min(bias_corrected, n = 1)` picks whichever `(model)` achieved the lowest `bias_corrected`. The `left_join` can create `NAs` for `best_model_candidate`/`best_model_bias` if a county has no model with `quality_ratio < 1`. Hence, `is_better` then checks two things at once: that a candidate exists (`!is.na(...)`) and that it performs better than the uncorrected baseline (`best_model_bias < bias_base`), because a model could `quality_ratio <1` yet still classifies further from Census than doing nothing (baseline). If both check fails, `best_model`/`best_bias` fall back to `"Baseline"`/`bias_base`, so every county in `best_per_county` gets the actual best-achievable outcome, which can be no correction at all. Then, we save it as a csv to use in `13_apply_best_model.R`.
