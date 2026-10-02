# `05_Availability_Census.R`

## Inputs
- `data/qs.census2012.txt`: the raw USDA NASS QuickStats bulk data export for the 2012 Census of Agriculture, containing every county-level statistic QuickStats publishes for that year, across all commodities and statistic categories. 
- `data/county_lookup.csv`: the `GEOID`/`STATEFP`/`NAME`/`LSAD` lookup table (produced by `00_setup_and_clip.R`), used here as the reference list of counties the analysis needs Census data for.
- `data/SF/states_2016.rds`: `sf` object of US state boundaries, used here only to re-derive the list of contiguous-US state FIPS codes.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `"<VERSION>"`): the CDL-code-to-category-to-Census mapping metadata. Here we use it to pull the list of `Census Commodity` values already assigned to some CDL code, so their Census-side availability can be checked.

## Outputs
None (console only). This script's real "output" is human judgment: it informs which lines get entered into the `Census Short_desc` column of `CDL_CENSUS_MAP_meta.xlsx`.

## Summary
Before trusting that a given CDL-mapped commodity's Census acreage figure is correct, this script answers two questions by hand: (1) is this commodity actually reported at the county level in the 2012 Census at all, for every county in the study area, and (2) when USDA QuickStats reports a commodity under several different `short_desc` lines (e.g. `CORN, GRAIN` and `CORN, SILAGE`), which of those lines should we summed together to correctly reconstruct that commodity's total acreage without double-counting. It works through the raw QuickStats bulk file, and we filter it down to the county-level relevant to this project. Then, we checks which counties are simply absent from Census reporting altogether (mostly independent cities/DC, which have no real farmland to report), and then, commodity by commodity, prints every raw `short_desc` line available so we manually decide the rule for summing `short_desc` in the metadata excel file.

## Detailed explanation

```r
census <- fread(CENSUS_PATH, sep = "\t", quote = "")
```
Uses `data.table::fread()` rather than a `readr`/base R reader because the raw QuickStats bulk export is large (`fread()` is dramatically faster).

```r
availability <- census[
  SOURCE_DESC == "CENSUS" &
    AGG_LEVEL_DESC == "COUNTY" &
    YEAR == 2012 &
    DOMAIN_DESC == "TOTAL"
][, .(
  GEOID = sprintf("%02d%03d", STATE_FIPS_CODE, COUNTY_CODE),
  county_name = COUNTY_NAME, state_fips = STATE_FIPS_CODE,
  commodity = COMMODITY_DESC, statisticcat = STATISTICCAT_DESC,
  short_desc = SHORT_DESC, value = VALUE
)]
```
Filters the (much larger) raw bulk file down to exactly the records relevant here: `SOURCE_DESC == "CENSUS"` (we don't look at the annual Survey, a separate QuickStats data source), `AGG_LEVEL_DESC == "COUNTY"` (county-level rows only, excluding state/national aggregates), `YEAR == 2012` (this project's Census reference year), and `DOMAIN_DESC == "TOTAL"` (excluding QuickStats' breakdowns by e.g. farm size or operation type, which would otherwise multiply-count each commodity). `sprintf("%02d%03d", STATE_FIPS_CODE, COUNTY_CODE)` reconstructs the standard 5-digit `GEOID` (2-digit state FIPS + 3-digit county code, zero-padded) since QuickStats stores these as separate numeric fields rather than a combined string.

```r
county_lookup$GEOID_padded <- sprintf("%05d", as.integer(county_lookup$GEOID))
missing_geoids <- setdiff(county_lookup$GEOID_padded, unique(availability$GEOID))
length(missing_geoids)
county_lookup[county_lookup$GEOID_padded %in% missing_geoids, ]
```
`county_lookup$GEOID` is explicitly re-padded to 5 digits here to guarantee it matches `availability$GEOID`'s format before comparison. `setdiff()` then finds every county in the study area's lookup table that has zero Census 2012 records at all. Printing the matching rows of `county_lookup` (which includes `LSAD`) lets us confirm that these are essentially all independent cities and DC, which have no real farmland to report and so are reasonably excluded from any acreage-based analysis rather than treated as a data gap needing correction.

```r
availability_clean <- availability[GEOID %in% county_lookup$GEOID_padded]
uniqueN(availability_clean$GEOID)
unique(availability_clean$statisticcat)
```
Restricts `availability` down to only the counties actually in this project's scope. `uniqueN()` (a `data.table` convenience for `length(unique(...))`) is a quick sanity check on how many distinct counties remain. Printing `unique(availability_clean$statisticcat)` surfaces every distinct statistic category QuickStats reports (harvested area, bearing area, etc.). We use this to decide which of these actually represent "acreage" for the next filtering step.

```r
keep_stats <- c("AREA HARVESTED", "AREA BEARING", "AREA NON-BEARING",
                "AREA BEARING & NON-BEARING", "AREA IN PRODUCTION",
                "AREA GROWN", "AREA NOT HARVESTED", "AREA")
availability_clean <- availability_clean[statisticcat %in% keep_stats]
```
Narrows to only the statistic of acreage we retain. We also keep some rarer statistic like `AREA GROWN` to ensure we don't remove commodity where it's the only available measure.

```r
available_bearing <- availability_clean[statisticcat == "AREA BEARING"]
available_bea_nonbea <- availability_clean[statisticcat == "AREA BEARING & NON-BEARING"]
intersect(unique(available_bearing$commodity), unique(available_bea_nonbea$commodity))
setdiff(unique(available_bearing$commodity), unique(available_bea_nonbea$commodity))
setdiff(unique(available_bea_nonbea$commodity), unique(available_bearing$commodity))
```
For some crops (orchards, vineyards), the Census sometimes reports "bearing" (mature, fruit-producing) acreage separately from "bearing & non-bearing" (all planted acreage regardless of maturity), and a commodity might only be reported under one of the two labels. This block checks, commodity by commodity, whether every commodity available under `AREA BEARING` is also available under the broader `AREA BEARING & NON-BEARING`, and whether any commodity is available under `AREA BEARING & NON-BEARING` but not `AREA BEARING` (the second `setdiff()` finds `Orchards` acreage is only reported in the broader "bearing & non-bearing" form). The conclusion is that it's safe to standardize on `AREA BEARING & NON-BEARING` wherever available, since it's always at least as complete as `AREA BEARING` alone.

```r
cdl_census_map <- read_excel(METADATA_XLSX, sheet = VERSION)
census_commodities_in_map <- cdl_census_map %>%
  pull(`Census Commodity`) %>% na.omit() %>% unique() %>% sort()

for (com in census_commodities_in_map){
  print(unique(availability_clean[commodity==com & statisticcat == "AREA HARVESTED"]$short_desc))
}
for (com in census_commodities_in_map){
  print(unique(availability_clean[commodity==com & statisticcat == "AREA BEARING & NON-BEARING"]$short_desc))
}
```
Pulls the distinct list of Census commodities the metadata excel has already assigned to some CDL code (`na.omit()` drops CDL codes with no Census counterpart), then loops over every one of those commodities and prints every distinct `short_desc` line QuickStats reports for it, separately under `AREA HARVESTED` and under `AREA BEARING & NON-BEARING`. It shows, for any commodity, every raw line available to sum. Everything below this point in the script (the `has_aggregated` check and the long list of per-commodity manual blocks) is used for convenience, it basically just filters that same information down to which commodities lack a single aggregated line and therefore need us to pick a summation rule.

```r
check_coverage <- function(stat_cats, label) {
  relevant <- availability_clean[
    statisticcat %in% stat_cats &
      grepl("ACRES HARVESTED$|ACRES BEARING & NON-BEARING$|ACRES IN PRODUCTION$", short_desc) &
      !grepl("IRRIGATED", short_desc)
  ]
  missing <- setdiff(census_commodities_in_map, unique(relevant$commodity))
  cat("\n", label, "\n")
  if (length(missing) == 0) cat("All covered\n")
  else cat("Missing (", length(missing), "):\n", paste0("  - ", missing, "\n"))
}
```
This restricts to `short_desc` lines actually ending in a relevant acreage phrase (`grepl(..., "$")` anchors the pattern to the end of the string so we don't match an unrelated line that contains one of those phrases mid-string) and explicitly excludes any `IRRIGATED` line as irrigated acreage is a subset of total acreage and summing it in would double-count). It then reports, by name, which of the mapped commodities have no matching line at all under the given statistic categories.

```r
area_stats <- availability_clean[
  statisticcat %in% c("AREA HARVESTED", "AREA BEARING & NON-BEARING", "AREA IN PRODUCTION") &
    grepl("ACRES HARVESTED$|ACRES BEARING & NON-BEARING$|ACRES IN PRODUCTION$", short_desc) &
    !grepl("IRRIGATED", short_desc)
]

length(setdiff(unique(county_lookup$GEOID_padded), unique(area_stats$GEOID))) #44 ? 6 more, which ones ?
missing_county_areastat <- setdiff(unique(county_lookup$GEOID_padded), unique(area_stats$GEOID))
county_lookup[county_lookup$GEOID_padded %in% missing_county_areastat &
              !county_lookup$GEOID_padded %in% missing_geoids, ]
```
Builds `area_stats`, the final cleaned acreage-only table. The following lines catch a second kind of coverage gap for county with some Census records but none in any of the relevant acreage statistic categories. We find 6 additional counties beyond the previously-known 38 independent cities/DC.

```r
geoid8111 <- census[census$STATE_FIPS_CODE == 8 & census$COUNTY_CODE == 111]
geoid26083 <- census[census$STATE_FIPS_CODE == 26 & census$COUNTY_CODE == 083]
unique(geoid26083$STATISTICCAT_DESC)
```
Manual, checks on two of those specific counties confirming that one county has only a total-land-area measure (not cropland-specific) and the other has no acreage-relevant statistic category reported at all. 

```r
has_aggregated <- sapply(target_commodities, function(com) {
  patterns <- paste0("^", com, " - ACRES (HARVESTED|BEARING & NON-BEARING)$")
  any(grepl(patterns, area_stats[commodity == com, short_desc]))
})

for (com in target_commodities[!has_aggregated]) {
  cat(" -", com, "\n")
}
```
For each mapped commodity, checks whether a `short_desc` exists that is exactly of the form `"<COMMODITY> - ACRES HARVESTED"` or `"<COMMODITY> - ACRES BEARING & NON-BEARING"`, meaning a single line whose commodity name matches the aggregate commodity name exactly. In this case, QuickStats already provides one total with nothing to sum. Printing the commodities that fail this check (`!has_aggregated`) produces the list for the manual, per-commodity check.

```r
#BEANS
unique(area_stats[commodity == "BEANS", short_desc])
#SUM:
# "BEANS, SNAP - ACRES HARVESTED"
# "BEANS, GREEN, LIMA - ACRES HARVESTED"
# "BEANS, DRY EDIBLE, (EXCL LIMA) - ACRES HARVESTED"
# "BEANS, DRY EDIBLE, LIMA - ACRES HARVESTED"
...
```
The remainder of the script is a sequence of nearly identical blocks, one per commodity lacking a single aggregated line (BEANS, BLUEBERRIES, CABBAGE, CHERRIES, CORN, CUT CHRISTMAS TREES, GREENS, HERBS, MELONS, MILLET, MINT, MUSTARD, ONIONS, PEAS, PEPPERS, POPCORN, SORGHUM, SUGARCANE, TOMATOES, WALNUTS): each prints every distinct `short_desc` QuickStats reports for that commodity, then we write as a comment the decision of exactly which subset of those lines should be summed to reconstruct the commodity's total acreage. In several cases, a judgment call to exclude a line. The PEAS block is the clearest example of this: it explicitly excludes `"PEAS, DRY, SOUTHERN (COWPEAS) - ACRES HARVESTED"` with the reasoning that CDL is observed to underestimate against it, concluding it's effectively a subset already captured by `"PEAS, DRY EDIBLE - ACRES HARVESTED"` and would double-count if included. These per-commodity decisions are exactly what we manually write into the `Census Short_desc` column of `CDL_CENSUS_MAP_meta.xlsx`, which `05bis` then reads and sums mechanically, with no further judgment involved at that stage.
