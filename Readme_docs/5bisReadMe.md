# `05bis_census_area.R`

## Inputs
- `data/qs.census2012.txt`: the raw USDA NASS QuickStats bulk data export for the 2012 Census of Agriculture.
- `data/county_lookup.csv`: the `GEOID`/`STATEFP`/`NAME`/`LSAD` lookup table (produced by `00_setup_and_clip.R`).
- `data/SF/states_2016.rds`: `sf` object of US state boundaries, used only to (re-)derive the contiguous-US state FIPS list.
- `data/metadata/CDL_CENSUS_MAP_meta.xlsx` (sheet `"<VERSION>"`): the CDL-to-category-to-Census mapping metadata. This script reads `Has Census`, `Census Commodity`, and the key input `Census Short_desc`, which by this point already holds our decisions (the ` + ` joined list of QuickStats lines to sum per commodity).

## Outputs
- `data/metadata/<VERSION>/census_shortdesc_map.csv`: the CDL-to-Census mapping "unpacked" into one row per `(commodity, short_desc)` pair: every commodity with `Has Census == 1`, with its ` + ` joined `Census Short_desc` string split into individual rows, one per QuickStats line that must be summed for that commodity.
- `data/CENSUS/<VERSION>/census_area_2012.csv`: the actual 2012 Census acreage data, filtered down to exactly the `short_desc` lines named in the mapping above, one row per `(GEOID, commodity-related short_desc)` observation, ready to be summed by commodity downstream.
- `data/metadata/<VERSION>/missing_geoid_census.csv`: every county in `county_lookup` that has no 2012 Census records at all, copied from `county_lookup` with an added `reason = "Not in Census"` column (mostly independent cities/DC).

## Summary
This is the production counterpart to the exploratory `05_Availability_Census.R`. It takes the mapping decisions written in `CDL_CENSUS_MAP_meta.xlsx`'s `Census Short_desc` column and turns them into two tables: a commodity-to-short_desc lookup, and the actual filtered Census acreage rows. Additionally, it produces a bookkeeping file of counties not within the Census. Everything here is version-dependent (`VERSION <- "v1"` or `"v2"`, selecting the corresponding sheet of the metadata excel).

## Detailed explanation

```r
census <- fread(CENSUS_PATH, sep = "\t", quote = "")
availability <- census[
  SOURCE_DESC == "CENSUS" & AGG_LEVEL_DESC == "COUNTY" &
    YEAR == 2012 & DOMAIN_DESC == "TOTAL"
][, .(GEOID = sprintf("%02d%03d", STATE_FIPS_CODE, COUNTY_CODE), ...)]

county_lookup <- read.csv(COUNTY_LOOKUP_PATH, stringsAsFactors = FALSE)
county_lookup$GEOID_padded <- sprintf("%05d", as.integer(county_lookup$GEOID))

availability_clean <- availability[GEOID %in% county_lookup$GEOID_padded]
keep_stats <- c("AREA HARVESTED", "AREA BEARING", ...)
availability_clean <- availability_clean[statisticcat %in% keep_stats]

missing_geoids <- setdiff(county_lookup$GEOID_padded, unique(availability$GEOID))
```
Identical loading/cleaning logic to `5_Availability_Census.R`.

```r
cdl_census_map <- read_excel(METADATA_XLSX, sheet = VERSION)

census_shortdesc_map <- cdl_census_map %>%
  filter(`Has Census` == 1) %>%
  mutate(`Census Commodity` = trimws(`Census Commodity`)) %>%
  select(commodity = `Census Commodity`, short_desc = `Census Short_desc`) %>%
  distinct() %>%
  separate_longer_delim(short_desc, delim = " + ") %>%
  mutate(short_desc = trimws(short_desc))

write.csv(census_shortdesc_map, SHORTDESC_MAP_PATH, row.names = FALSE)
```
This is the step that applies the mapping decisions. `filter(Has Census == 1)` keeps only CDL codes/commodities that actually have a Census counterpart. `distinct()` collapses duplicate `(commodity, short_desc)` pairs since several different CDL codes can map to the same `Census Commodity` (e.g. multiple corn-related CDL codes all pointing to `CORN`), this avoids carrying forward redundant rows. `separate_longer_delim(short_desc, delim = " + ")` is the key unpacking step: it splits a single cell like `"CORN, GRAIN - ACRES HARVESTED + CORN, SILAGE - ACRES HARVESTED"` into two separate rows, one per `' + '` joined line. `trimws()` on both the commodity name and the split `short_desc` values guards against leading/trailing whitespace that would otherwise break an exact-string match against the Census data downstream.

```r
area_stats <- availability_clean[
  statisticcat %in%
    c("AREA HARVESTED", "AREA BEARING & NON-BEARING", "AREA IN PRODUCTION", "AREA GROWN") &
    grepl("ACRES HARVESTED$|ACRES BEARING & NON-BEARING$|ACRES IN PRODUCTION$|ACRES GROWN$", short_desc) &
    !grepl("IRRIGATED", short_desc)
]

selected_descs <- census_shortdesc_map$short_desc

census_area <- area_stats[short_desc %in% selected_descs] %>%
  mutate(GEOID = sprintf("%05d", as.integer(GEOID)))

missing_county_shortdesc <- setdiff(unique(county_lookup$GEOID_padded), unique(census_area$GEOID))

write.csv(census_area, CENSUS_AREA_PATH, row.names = FALSE)
```
`area_stats` applies the same filtering pattern used in `05_Availability_Census.R`'s `check_coverage()`, producing a clean table of all acreage-relevant Census rows regardless of commodity. `census_area <- area_stats[short_desc %in% selected_descs]` is the actual filtering step. `GEOID` is re-padded to 5 digits again to be sure.

```r
census_missing <- county_lookup[county_lookup$GEOID_padded %in% missing_geoids, ] %>%
  mutate(reason = "Not in Census")

write.csv(census_missing, CENSUS_MISSING_PATH, row.names = FALSE)
```
Writes out the file of counties absent from the Census entirely.
