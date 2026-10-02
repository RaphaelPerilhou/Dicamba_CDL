# `5ter_Hay_exploration.R`

## Inputs
- `data/qs.census2012.txt`: raw 2012 Census of Agriculture QuickStats bulk export.
- `data/county_lookup.csv`: `GEOID`/`STATEFP` lookup table.
- `outputs/v1/classification_summary_disagg_wide.csv`: the wide-format disaggregated CDL pixel counts by code (produced by `01bis_classification_dataframes.R`, `VERSION` = `"v1"` here since it needs total historical HAY/Alfalfa/Other-Hay/Clover/Vetch acreage regardless of which mapping version is currently authoritative).

## Outputs
None (console only).

## Summary
This script produces the sanity check that justifies splitting Alfalfa out of the aggregated "HAY" Census commodity and dropping Other Hay/Clover/Vetch from the analysis. It does two things: (1) checks whether summing county-level Alfalfa-specific Census acreage up to the state level roughly matches the state-level QuickStats total directly (the gap being acreage suppressed at the county level for disclosure reasons), confirming Alfalfa can be reliably isolated from the rest of HAY; and (2) prints the total CDL pixel-derived acreage for each of the four hay-related CDL codes (36 Alfalfa, 37 Other Hay, 58 Clover/Wildflowers, 224 Vetch) for 2012, to have an idea of how much acreage would be gained or lost by keeping vs. dropping each one.

## Detailed explanation

```r
alfalfa_state <- availability_clean[commodity == "HAY"] %>%
  filter(short_desc == "HAY, ALFALFA - ACRES HARVESTED") %>%
  group_by(state_fips) %>%
  summarise(state_total = sum(as.numeric(gsub(",", "", value)), na.rm = TRUE))
```
Sums county-level Alfalfa acreage up to state totals. `gsub(",", "", value)` strips thousands-separator commas from the raw QuickStats value strings before coercing to numeric (QuickStats formats large numbers like `"1,234,567"` as text). `na.rm = TRUE` here means counties with a suppressed/missing value (QuickStats withholds figures that could identify an individual farm) are simply treated as contributing 0 to the state sum.

```r
availability_states_clean <- ... %>% filter(short_desc == "HAY, ALFALFA - ACRES HARVESTED")
QS_states <- availability_states_clean %>%
  group_by(state_fips) %>%
  summarise(state_total = sum(as.numeric(gsub(",", "", value)), na.rm = FALSE))
```
Same commodity/short_desc filter, but pulled directly from QuickStats' own `AGG_LEVEL_DESC == "STATE"` records rather than summed up from counties. `na.rm = FALSE` here because state-level figures are essentially never suppressed (less likely to have a single farm in a whole state compared to a county), so an `NA` would signal something unexpected (a missing state row) rather than a disclosure suppression.

```r
cat("Gap:", sum(QS_states$state_total) - sum(alfalfa_state$state_total), ...)
#The gap can be seen as the total acres suppressed for disclosure.
```
Comparing the two totals: the state-level QuickStats figure minus the county-summed figure isolates exactly the acreage QuickStats declined to report at the county level. A small gap (relative to the total) supports treating county-level Alfalfa data as reliable enough to use.

```r
CDL_crops <- read_csv(paste0("outputs/", VERSION, "/classification_summary_disagg_wide.csv")) %>%
  filter(year == 2012) %>%
  select(statefp, geoid, year, `36`, `37`, `58`, `224`)

cat("CDL Alfalfa (36) acres:      ", sum(CDL_crops$`36`)  * PIXEL_ACRES, "\n")
...
```
Pulls 2012 per-code pixel counts directly from the disaggregated wide table (one column per raw CDL code, produced earlier in the pipeline) for the four hay-related codes, and converts each to acres via the standard `PIXEL_ACRES` conversion factor. These four printed totals are what let us directly compare how much acreage each code contributes to keep Alfalfa on its own but exclude Other Hay, Clover/Wildflowers, and Vetch as too noisy relative to their size.
