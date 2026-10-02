# CDL_CENSUS_MAP metadata

## Goal
Single source of truth for: (1) which CDL codes are used in the Dicamba classification analysis, (2) which category (GM / Tolerant / Vulnerable / NonCrop / Unclassified) each (used) CDL code maps to, and (3) which USDA Census of Agriculture short_desc line(s) each mapped commodity uses.

## Sheets
- **v1**: FROZEN historical record (Never edit this sheet again) as it documents how the original (pre-Alfalfa-remapping) results were produced.
- **v2**: CURRENT authoritative. Same as v1 but remove the following CDL codes of the analysis: 37 (Other Hay/Non-Alfalfa), 58 (Clover/Wildflowers), and 224 (Vetch). Also, CDL code 36 (Alfalfa) now maps to the Census short_desc 'HAY, ALFALFA - ACRES HARVESTED' instead of the aggregated 'HAY - ACRES HARVESTED' line. The goal was to remove the very noisy HAY commodity while keeping the Alfalfa which matter in the Dicamba Analysis.

Every script in the pipeline that needs a CDL-code-to-category or CDL-code-to-Census mapping reads **v2**. **v1** is kept only as an archival record of exactly what produced the earlier (pre-Alfalfa-remapping) results, in case those need to be reproduced or audited later. Both sheets share an identical column layout, described below.

## Column definitions

| Column | Meaning |
|---|---|
| **CDL Code** | The NASS Cropland Data Layer numeric class code. |
| **CDL Label** | The official CDL legend description for that code. |
| **Keep** | `1` if this code is included in the current analysis, `0` if not. Rows with `Keep = 0` are shaded grey for readability. To reuse this pipeline for a different set of CDL codes, set differents codes to 1 and 0 in this column  |
| **Category Code** / **Category** | `0` = NonCrop, `1` = GM-enabled, `2` = Tolerant, `3` = Vulnerable. Blank means the code is not Kept, OR the code is Kept but should fall through to the Unclassified (99) default described below. |
| **Census Commodity** | The USDA QuickStats `COMMODITY_DESC` this CDL code is validated against. Blank if this code has no Census counterpart. |
| **Has Census** | `1` if a Census Commodity is assigned, `0`/blank otherwise. |
| **Census Short_desc** | The EXACT QuickStats `SHORT_DESC` line(s) summed to get that commodity's acreage. Multiple lines joined with `' + '` are all summed together. This column is what the script reads directly to assign the correct short_desc. |
| **Notes / Ambiguity** | Free-text rationale, especially for any code whose Census match required a judgment call. |

## Codes NOT covered by either sheet (Keep is implicitly 0)

- A pixel enters the union ag mask if it carried ANY `Keep = 1` code in AT LEAST ONE year of the study period.
- Once in the mask, a pixel with CDL code 61 (Fallow/Idle) that year gets category 0 (NonCrop).
- A pixel in the mask whose CDL code that year is `Keep = 1` gets that code's Category.
- A pixel in the mask whose CDL code that year is EITHER not in this workbook at all, OR is listed with `Keep = 0`, gets category 99 (Unclassified) for that year (e.g. a pixel that was Soybeans in 2012 (entering the mask) but Forest in 2009 gets 99 in 2009).

## Double-crop priority rule

When a CDL double-crop code combines components from different categories, the single Category value already recorded for that code is chosen according to this rule: GM-enabled (1) > Tolerant (2) > Vulnerable (3). E.g. code 26 (Dbl Crop WinWht/Soybeans) is GM-enabled because Soybeans outranks Winter Wheat; code 231 (Dbl Crop Lettuce/Cantaloupe) is Vulnerable because both components are. Do not "re-derive" a double-crop code's category from its components, the value recorded in *Category* is authoritative.

See `docs/context/double_cropping.md` for the full explanation.
