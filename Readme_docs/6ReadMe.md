# `06_census_acres.R`

## Inputs
- `data/CENSUS/<VERSION>/census_area_2012.csv`: filtered Census acreage rows, one per `(GEOID, short_desc)` (produced by `05bis`).
- `data/county_lookup.csv`: `GEOID`/`STATEFP`/`NAME` lookup table.
- `data/metadata/<VERSION>/census_shortdesc_map.csv`: commodity-to-short_desc mapping.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `"<VERSION>"`): used here for its `Has Census`, `Census Commodity`, `Category`, `Category Code` columns, to attach a category to each commodity.

## Outputs
- `outputs/<VERSION>/commodity_census_acres_2012.csv`: one row per `(GEOID, commodity)`, with summed acres and suppression counts.
- `outputs/<VERSION>/category_census_acres_2012.csv`: one row per `(GEOID, Category)`, acres and suppression counts further summed to the GM/Tolerant/Vulnerable/NonCrop category level.
- `outputs/<VERSION>/census_suppressed_log.csv`: every individual `(GEOID, commodity, short_desc)` row QuickStats disclosure-suppressed, for audit.

## Summary
This script turns the raw, line-level Census acreage data into the two acreage tables, one at the commodity level, and one collapsed to category level. It also handles QuickStats' disclosure codes correctly to distinguish a genuine, small-but-known value (`(Z)`, "less than half the unit shown," treated as a real 0) from a value withheld to protect an individual farm's privacy (`(D)`, true suppression, also zero-filled but flagged so it isn't mistaken for a confirmed zero).

## Detailed explanation

```r
census_area <- census_area %>%
  mutate(
    value = gsub(",", "", value),
    is_Z = grepl("\\(Z\\)", value),
    is_D = grepl("\\(D\\)", value),
    suppressed = is_D,
    value_clean = case_when(
      is_Z ~ 0,
      is_D ~ 0,
      grepl("^[0-9]+$", value) ~ as.numeric(value),
      TRUE ~ NA_real_
    )
  )
```
Classifies each raw value: `(Z)` becomes a real, known `0`; `(D)` also becomes `0` but is flagged via `suppressed`. Anything that isn't numeric and isn't `(Z)`/`(D)` becomes `NA`.

```r
agg_commodity <- census_area %>%
  group_by(GEOID, commodity) %>%
  summarise(
    acres = sum(value_clean, na.rm = TRUE),
    n_suppressed = sum(suppressed),
    n_total = n(),
    fully_suppressed = n_suppressed == n_total,
    .groups = "drop"
  )
```
Sums the cleaned acreage across every `short_desc` line belonging to the same commodity (e.g. summing Bell Pepper + Chile Pepper into PEPPERS), and tracks, alongside the total, how many of that commodity's lines were suppressed and whether all of them were. `fully_suppressed` matters because a commodity's acreage of `0` after summing is ambiguous unless you know whether that's a true zero or every contributing line was removed.

```r
commodity_category <- cdl_census_map %>%
  filter(`Has Census` == 1) %>%
  mutate(`Census Commodity` = trimws(`Census Commodity`)) %>%
  select(`Census Commodity`, Category, `Category Code`) %>%
  distinct()

commodity_category <- agg_commodity %>%
  left_join(commodity_category, by = c("commodity" = "Census Commodity"))

census_category <- commodity_category %>%
  group_by(GEOID, `Category Code`, Category) %>%
  summarise(
    acres_census = sum(acres, na.rm = TRUE),
    n_suppressed = sum(n_suppressed),
    .groups = "drop"
  )
```
Attaches each commodity's category (GM/Tolerant/Vulnerable/NonCrop) from the metadata excel file, then re-aggregates from commodity level up to category level per county, summing both acres and suppression counts across every commodity sharing a category. This produces the category-level Census acreage table that gets compared directly against CDL's own category-level pixel-derived acreage.

```r
n_fully_suppressed <- sum(agg_commodity$fully_suppressed)
n_partially_suppressed <- sum(agg_commodity$n_suppressed > 0 & !agg_commodity$fully_suppressed)
```
A final reporting step distinguishing two very different situations among county-commodity pairs with any suppression at all: fully suppressed (every contributing line removed) versus partially suppressed (some lines removed).
