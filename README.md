# Madagascar Habitat Prediction

## Project Overview

**Research question:** How has forest loss and fragmentation in Madagascar (2000–2025) been associated with human development, and what are the potential consequences for selected lemur species under future development scenarios, predicted using a machine learning model?

**Approach:** Build a spatial panel dataset at 10km grid-cell resolution, tracking forest extent, fragmentation, and human-development pressures (roads, population, agriculture, night lights, urbanisation) over time. Use this to statistically characterize the relationship between development and deforestation, then model lemur habitat suitability and forecast future impact under development scenarios.

**Spatial unit of analysis:** 10km × 10km grid cells, clipped to Madagascar's coastline — 7,036 cells total.
**Temporal unit of analysis:** Nominal panel years 2000, 2005, 2010, 2015, 2020, 2024 (not all variables have data at every year — see per-phase notes).

---

## Repository Structure

```
Madagascar habitat prediction/
├── data/
│   ├── hansen/
│   │   ├── treecover2000/   — baseline canopy cover, 4 raw tiles + processed
│   │   ├── lossyear/        — year of deforestation per pixel, 4 raw tiles + processed
│   │   └── gain/             — regrowth 2000–2012, 4 raw tiles + processed
│   ├── raw/                 — reserved for population, agriculture, night lights source rasters
│   └── processed/
│       ├── forest_masks/                     — binary forest/non-forest rasters, one per year
│       ├── madagascar_boundary_projected.gpkg
│       ├── madagascar_grid_10km.gpkg
│       ├── madagascar_grid_clipped.gpkg      — spatial unit of analysis (7,036 cells; id/grid_id field)
│       └── statistics/
│           ├── forest_by_grid_year.parquet / .csv / .xlsx        — grid_id, year, forest_pct, forest_pixels (42,216 rows = 7,036 cells × 6 years) ✅ verified
│           ├── fragmentation_by_grid_year.parquet / .csv          — grid_id, year, patches, edge (42,216 rows = 7,036 cells × 6 years) ✅ verified
│           ├── master_forest_fragmentation.parquet                — forest + fragmentation merged
│           ├── roads_by_grid_year.parquet / .xlsx                 — grid_id, year, road_length_m, road_density_km_per_km2 (2010/2015/2020/2024 only)
│           ├── roads_mapping_effort_by_grid_year.parquet          — diagnostic only: highway_contributor_count per cell per interval
│           ├── roads_review.xlsx                                  — output of export_roads_to_excel.py
│           ├── road_growth_hotspots.gpkg                          — QGIS layer: road growth by cell/period, for visual QA
│           ├── population_by_grid_year.parquet / .xlsx             — grid_id, year, source_year, population_sum (2000/2005/2010/2015/2020/2024) ✅ verified
│           ├── agriculture_by_grid_year.parquet / .xlsx             — grid_id, year, source_year, source_dataset, cropland_pct (2000/2005/2010/2015/2020/2024) ✅ verified
│           └── nightlights_by_grid_year.parquet / .xlsx             — grid_id, year, source_type, mean_radiance (2000/2005/2010/2015/2020/2024) ✅ verified
├── data/raw/worldpop/          — cached WorldPop global rasters (local download, one per year, ~1GB each)
├── data/raw/agriculture/glad/       — cached GLAD global cropland tiles (1 file per epoch, SE quadrant only)
├── data/raw/agriculture/worldcover/ — cached WorldCover 3°×3° tiles + mosaic.tif, one subfolder per year (2020, 2021)
├── data/raw/nightlights/        — cached harmonized night-lights zips + extracted tifs, one subfolder per panel year
├── qgis/                      — QGIS project files, visual QA only
├── packages/, venv/           — environment
└── madagascar_habitat.qgz     — QGIS project (visualization/inspection, not processing)
```

### Scripts produced so far (all currently sit in the project root — moving them into a `scripts/` folder is still the recommended next housekeeping step)

> Note: filenames reconciled against the actual project folder listing. `test.py` consolidates what earlier notes referred to as three separate diagnostic scripts (`check_road_growth_concentration.py`, `check_effort_vs_growth_concentration.py`, `check_changeset_metadata.py`) — kept as one file, not three. `roads_to_excel.py` is the stale/earlier version of `export_roads_to_excel.py`, which is the current script and the one that produces `roads_review.xlsx`. `join_tables.py` is still unconfirmed — likely an early pass at the Phase 13 master merge.

| Script | Purpose |
|---|---|
| `Mosaic_and_clip.py` | Mosaics Hansen tiles and clips to the Madagascar boundary (Phase 4) |
| `reproject.py` | Reprojects clipped rasters to EPSG:3857 to match the grid (Phase 4) |
| `forest_masks.py` | Derives per-year binary forest masks from treecover2000 + lossyear + gain (Phase 5–6) |
| `zonal_stats.py` / `zonal_stats_v2.py` | Computes % forest cover per grid cell per year → `forest_by_grid_year` (Phase 6) |
| `fragmentation.py` | Computes patch count + edge length per grid cell per year via pylandstats → `fragmentation_by_grid_year` (Phase 7) |
| `OSM_roads.py` | Pulls time-varying road length from the ohsome API (Phase 8) |
| `export_roads_to_excel.py` | Exports `roads_by_grid_year` to Excel, incl. `roads_review.xlsx` (Phase 8, current) |
| `roads_to_excel.py` | Superseded by `export_roads_to_excel.py` — candidate for deletion |
| `growth_hotspots.py` | Exports per-cell road growth as a GeoPackage layer for QGIS (Phase 8) |
| `test.py` | Consolidates the road-growth-concentration, mapping-effort, and changeset-metadata diagnostics from the Phase 8 investigation |
| `WorldPop_population.py` | Downloads WorldPop annual population rasters locally and zonal-sums onto the grid → `population_by_grid_year.parquet` (Phase 9, **complete**) |
| `export_population_to_excel.py` | Exports `population_by_grid_year` to Excel, mirroring the roads export (Phase 9, current) |
| `GLAD_agriculture.py` | Downloads GLAD cropland epochs + ESA WorldCover tiles, computes % cropland per cell → `agriculture_by_grid_year.parquet` (Phase 10, **complete**) |
| `agriculture_to_excel.py` | Exports `agriculture_by_grid_year` to Excel, mirroring the roads/population exports, incl. the GLAD cumulative-check verdict (Phase 10, current) |
| `srunet_nightlights.py` | Downloads the harmonized DMSP/VIIRS night-lights dataset, computes mean radiance per cell → `nightlights_by_grid_year.parquet` (Phase 11, **complete**) |
| `export_nightlights_to_excel.py` | Exports `nightlights_by_grid_year` to Excel, mirroring the roads/population/agriculture exports (Phase 11, current) |
| `join_tables.py` | Confirmed: an early, incomplete pass at the Phase 13 master merge — currently only joins forest + fragmentation, predates roads/population/agriculture. Do not treat as the final Phase 13 merge. |

---

## Pipeline: Completed Work

### Phase 1–2 — Project Setup, Boundary & Grid
Defined the Madagascar boundary and built a 10km fishnet grid, clipped to the coastline (`madagascar_grid_clipped.gpkg`, 7,036 cells). This is the fixed spatial unit every later variable is aggregated into.

### Phase 3 — Forest Data Acquisition
Downloaded Hansen Global Forest Change v1.12 tiles (4 tiles covering Madagascar) for three variables: `treecover2000` (2000 baseline % canopy cover), `lossyear` (year of loss, 2001–2024), `gain` (binary regrowth, 2000–2012 only, no per-year attribution).

### Phase 4 — Preprocessing
Mosaicked each variable's 4 tiles, clipped to the Madagascar boundary, reprojected to EPSG:3857 to match the grid.
**Bug resolved:** an initial clip silently failed (output raster was the full unclipped tile rectangle, including Comoros/Mauritius) — fixed by force-clipping the final output array rather than trusting upstream inputs.

### Phase 5–6 — Forest Area & Change (Derivation) ✅ Verified
Hansen doesn't provide a direct per-year forest layer, so one was derived: a pixel counts as forest in year Y if it was ≥25% canopy in 2000 AND hasn't yet appeared in `lossyear` by year Y, with `gain` pixels added back in from 2013 onward (documented simplification — `gain` has no per-year timestamp within its 2000–2012 window). Produced `forest_masks/forest_{year}.tif` for 2000, 2005, 2010, 2015, 2020, 2024.

**Performance issue resolved:** initial rasters were untiled/uncompressed (1-pixel-tall row strips), making zonal stats impractically slow — fixed via tiled, ZSTD/deflate-compressed GeoTIFF writes.
**Considered, deferred:** JRC TMF as a supplementary dataset to cover regrowth beyond 2012 — marginal gain judged not worth the added complexity.

**Zonal statistics:** % forest cover per grid cell per year computed via rasterstats/exactextract → `forest_by_grid_year.parquet` (columns: `grid_id`, `year`, `forest_pct`, `forest_pixels`; 42,216 rows = 7,036 cells × 6 years, confirmed against the exported file). National trend verified as a smooth, plausible decline (~30.7% in 2000 → ~22.2% in 2024).

### Phase 7 — Fragmentation ✅ Verified
Patch count and total edge length per grid cell per year, computed via `pylandstats` directly on `forest_masks/forest_{year}.tif` (no new external data).
**Bugs resolved:** (1) wrong method name (`class_metrics` → `compute_class_metrics_df`); (2) a nodata/class-collision bug where pylandstats' default treatment of 0 as "nodata" silently zeroed out all edge-length calculations while leaving patch counts unaffected.
Output: `fragmentation_by_grid_year.parquet` (columns: `grid_id`, `year`, `patches`, `edge`; 42,216 rows = 7,036 cells × 6 years, confirmed against the exported file).

### Phase 8 — Roads ✅ Complete
Originally scoped as a static layer (assumption: "OSM has no historical archive"). This assumption was revisited and found to be only partially true.

**Method:** the ohsome API (HeiGIT) exposes OSM's full edit history and allows querying road length as of a past date. Pulled per grid cell for 2010, 2015, 2020, 2024 via `/elements/length/groupBy/boundary`.
**Hard limit found:** 2000 and 2005 are unavailable — OSM's tracked history only starts 2007-10-08, and ohsome rejects a multi-timestamp request entirely if any timestamp predates that (this caused an initial batch of confusing identical 404s across all cells).

**National summary** (n = 7,036 cells in every year):

| Year | Total road length (km) | Mean density (km/km²) | % change vs. prior period |
|---|---:|---:|---:|
| 2010 | 6,541.68 | 0.00981 | — |
| 2015 | 57,756.40 | 0.09565 | +782.9% |
| 2020 | 127,933.51 | 0.19867 | +121.5% |
| 2024 | 494,817.95 | 0.73432 | +286.8% |

**Anomaly investigated:** national road length grew sharply and non-monotonically across the four snapshots above. Investigation process:
1. **Concentration check** — growth is spatially concentrated in a small share of cells each period (e.g. 50% of 2015→2020 growth in just 4.2% of cells), increasingly so over time.
2. **Mapping-effort covariate** (unique contributors, all edit types) didn't track the growth pattern — ruled out a simple "more people mapping" explanation.
3. **Direct changeset inspection** (ohsome `/contributions/geometry`, then OSM's own Changesets API) revealed the true cause: manual JOSM edits by named volunteer mappers, organized under specific project hashtags — a geo-health research project (`#geohealthresearch-project-14`, tied to Institut Pasteur Madagascar / IRD / Pivot Madagascar) and Operation Fistula's Missing-Maps-style campaign (`#opfistula #projectfree`), concentrated around Andramasina, Analamanga, mid-2022.

**Conclusion:** not a bulk/bot import, but a **mapping-campaign-timing artifact** — health/humanitarian NGOs mapped these areas because they needed maps for service delivery, which has no necessary relationship to (and may anti-correlate with) local development level.

**Decision:** `roads_by_grid_year.parquet` is kept in full as the Phase 8 deliverable, but only the **2024 snapshot** will be joined into modelling — as a current-state road-network layer for Phase 18–21, not as a time-varying predictor in the Phase 14–17 regression.

**Bug resolved and confirmed (post-completion):** `road_density_km_per_km2` came back all-`NaN` due to a type mismatch — the area lookup dict was keyed by `grid_id` in its native (integer) dtype from the grid file, while ohsome returns `grid_id` as a string, so every dict/`.map()` lookup silently failed. Fixed by casting `grid_id` to string immediately after loading the grid, in every script that touches it. Re-run confirmed sane values: national density rose from 0.0098 km/km² (2010) to 0.7343 km/km² (2024), consistent with total length ÷ Madagascar's ~587,000 km² landmass.

Also produced `roads_mapping_effort_by_grid_year.parquet` (highway-specific contributor counts per interval) as a diagnostic/QA artifact — not part of the master table.

---

## Pipeline: Remaining Work

### Phase 9 — Population ✅ Complete
**Resource:** WorldPop unconstrained global population mosaics, 1km resolution, genuine annual coverage 2000–2020 (`https://data.worldpop.org/GIS/Population/Global_2000_2020/{year}/0_Mosaicked/ppp_{year}_1km_Aggregated.tif`). Unlike roads, this is a real per-year time series, not subject to a mapping-effort artifact.
**Known gap:** no true 2024 raster exists; the pipeline substitutes WorldPop's 2023 "interim" global mosaic, labeled `source_year=2023` so the substitution is traceable downstream.
**Method:** zonal **sum** (not mean — population is a count, like forest pixel counts) per grid cell. Originally attempted via GDAL's `/vsicurl/` streaming to avoid downloading full ~1.1GB global rasters; switched to full local download after `/vsicurl/` failed (see bug below).

**Bug resolved:** `/vsicurl/` streaming failed with `Range downloading not supported by this server!` — WorldPop's server doesn't reliably honor HTTP Range requests. Fixed by downloading each year's global raster fully to `data/raw/worldpop/` (cached, so a re-run skips already-downloaded years) and running zonal stats against the local file instead of streaming.

**Bug resolved:** first full run returned `inf`/`-inf` national totals for 2000–2020 (2024 alone came out correct). Cause: the script assumed a uniform `-99999` nodata sentinel across all WorldPop products, but the 2000–2020 annual mosaics use a different nodata value than the 2023 interim file; unmasked nodata pixels summed in as extreme values and overflowed float32. Fixed by reading each file's actual nodata value from its own raster metadata via `rasterio` rather than hardcoding it, plus a sanity clip that flags (as `NaN`, not silently) any per-cell sum outside a plausible range for a 100km² cell.

**Result — national totals, verified against UN WPP-based reference (n = 7,036 cells in every year):**

| Year | Total population (M) | Mean per-cell population | Source raster year | % change vs. prior | UN reference (M) | Diff vs. reference |
|---|---:|---:|---:|---:|---:|---:|
| 2000 | 15.29 | 2,173 | 2000 | — | 16.5 | −7.3% |
| 2005 | 17.36 | 2,467 | 2005 | +13.5% | 19.2 | −9.6% |
| 2010 | 19.98 | 2,840 | 2010 | +15.1% | 22.2 | −10.0% |
| 2015 | 23.34 | 3,317 | 2015 | +16.8% | 25.4 | −8.1% |
| 2020 | 27.68 | 3,933 | 2020 | +18.6% | 29.0 | −4.6% |
| 2024 | 30.08 | 4,275 | 2023 (interim) | +8.7% | 32.0 | −6.0% |

Trend is smooth and monotonic; the consistent −4% to −10% gap against the UN reference is typical of WorldPop's *unconstrained* product (which tends to undercount relative to UN estimates) rather than a pipeline defect. Output: `population_by_grid_year.parquet` (columns: `grid_id`, `year`, `source_year`, `population_sum`), exported to `population_by_grid_year.xlsx` via `export_population_to_excel.py` with `national_summary` / `by_cell` / `notes` sheets, mirroring the Phase 8 roads export.

### Phase 10 — Agriculture ✅ Complete
**Resource:** GLAD Global Cropland Extent (Potapov et al., glad.umd.edu), 30m, binary cropland/not-cropland — but not annual: shipped as five 4-year epochs labeled by end year (2003/2007/2011/2015/2019). Spliced with ESA WorldCover (esa-worldcover.org), 10m, 11-class land cover (cropland = class 40) — only two years exist, 2020 (v100) and 2021 (v200), nothing beyond.

**Year-alignment decision (made explicitly, before writing the join, per the original Phase 10 note):**

| Panel year | Source | Source year | Note |
|---|---|---:|---|
| 2000 | GLAD | 2003 | nearest epoch, ~3yr off |
| 2005 | GLAD | 2007 | nearest epoch, ~2yr off |
| 2010 | GLAD | 2011 | nearest epoch, ~1yr off |
| 2015 | GLAD | 2015 | exact |
| 2020 | WorldCover | 2020 | exact |
| 2024 | WorldCover | 2021 | nearest available, same substitution pattern as Phase 9's population 2023→2024 |

Every row carries `source_year` and `source_dataset` so the substitution is traceable downstream, same convention as Phase 9.

**Bug resolved:** WorldCover's own data-access page states tiles are "organized in a grid of 1 by 1 degrees" — this is wrong. Confirmed via a Microsoft Planetary Computer STAC example (tile `N00E033`) that the real grid is **3° × 3°**. Requesting 1°×1° tiles meant only ~6 of 75 requested tile codes happened to land on a real filename, silently leaving 5,229/7,036 cells (74%) with no WorldCover data at all in the first run. Fixed by snapping tile enumeration to the 3° grid — all 7,036 cells now resolve for both 2020 and 2024.

**Question resolved — GLAD cumulative vs. independent epochs:** GLAD's own documentation doesn't state whether its four epoch layers are cumulative-since-2000 or standalone snapshots per window — this matters because cumulative epochs would make cropland *loss* structurally unmeasurable (a pixel could only ever turn on). Checked empirically: **3,101 cell-transitions showed a decrease in `cropland_pct` between consecutive GLAD epochs** (e.g. grid_id 119: 2000=0.35% → 2005=0.27%). This rules out cumulative accounting — the epochs are independent per-epoch snapshots, so real cropland loss/abandonment is genuinely visible in this series and it can be used as a time-varying signal in Phase 14–17, not just an expansion-only trend.

**Result — national mean cropland % (n = 7,036 cells in every year):**

| Panel year | Mean cropland % | Source | % change vs. prior |
|---|---:|---|---:|
| 2000 | 2.05% | GLAD 2003 | — |
| 2005 | 2.24% | GLAD 2007 | +9.2% |
| 2010 | 2.39% | GLAD 2011 | +6.8% |
| 2015 | 2.78% | GLAD 2015 | +16.3% |
| 2020 | 5.20% | WorldCover 2020 | +87.4% |
| 2024 | 6.12% | WorldCover 2021 | +17.6% |

**Known limitation:** the 2015→2020 jump (2.78% → 5.20%, +87%) is almost certainly inflated by the GLAD→WorldCover methodology switch (30m binary mask → 10m 11-class land cover, different sensors/algorithms) rather than real cropland expansion of that magnitude in five years — treat this transition as a level-shift artifact, not a measured growth rate, the same way Phase 8's roads trend needed a mapping-campaign caveat. A same-source sensitivity check (e.g. comparing GLAD's 2019 epoch against WorldCover 2020 directly, both real, adjacent years) would help quantify how much of the jump is method vs. reality, before this feeds Phase 14–17.

Output: `agriculture_by_grid_year.parquet` (columns: `grid_id`, `year`, `source_year`, `source_dataset`, `cropland_pct`), exported via `export_agriculture_to_excel.py` with `national_summary` / `by_cell` / `notes` sheets (notes sheet includes the cumulative-check verdict computed directly from the data).

### Phase 11 — Night Lights ✅ Complete
**Resource:** replaced the originally-planned raw DMSP-OLS (pre-2013) + VIIRS DNB (2012–present) with manual intercalibration, per the literature-check recommendation in the original note. Used instead: Chen et al., ["A global annual simulated VIIRS nighttime light dataset from 1992 to 2023"](https://www.nature.com/articles/s41597-024-04228-6) (updated through 2024), *Scientific Data* (2024) — a single harmonized product covering 1992–2024 in one consistent unit, using a super-resolution model to reconstruct VIIRS-equivalent radiance for the pre-2012 DMSP years. **Every panel year has an exact match** (2000/2005/2010/2015/2020/2024) — no nearest-year substitution needed anywhere, unlike Phase 9/10. `source_type` distinguishes `simulated` (2000/2005/2010, DMSP-reconstructed) from `observed` (2015/2020/2024, real VIIRS).

**Bug resolved:** the same 16 grid cells returned no data in every year regardless of dataset. Diagnosed as rasterstats' default `all_touched=False` only counting a pixel if its *center* falls inside the polygon — for small coastal-sliver cells at this raster's ~500m resolution, that can yield zero qualifying pixels even though the polygon genuinely overlaps real data. Fixed with `all_touched=True`; confirmed via a diagnostic showing 0 cells still missing afterward, ruling out a genuine data gap.

**Bug resolved:** small negative `mean_radiance` values were initially (wrongly) treated as bad data and discarded to `NaN`. VIIRS-derived radiance has sensor/model noise centered near zero in genuinely dark, unlit areas — standard practice is to clip negative values to 0, not discard the cell. This was silently NaN'ing Madagascar's darkest rural cells (exactly what a low-electrification country should look like) until corrected.

**Result — national mean radiance (n = 7,036 cells in every year):**

| Panel year | Mean radiance | Source type | % change vs. prior |
|---|---:|---|---:|
| 2000 | 0.001793 | simulated | — |
| 2005 | 0.001708 | simulated | −4.8% |
| 2010 | 0.002380 | simulated | +39.4% |
| 2015 | 0.005159 | observed | +116.7% |
| 2020 | 0.006651 | observed | +28.9% |
| 2024 | 0.011877 | observed | +78.6% |

**Open question, not yet resolved:** the three simulated years are an order of magnitude smaller and noisier in direction (a slight *dip* 2000→2005, then a rise) than the clean, larger jumps in the observed years. Could be genuine (limited, non-monotonic electrification before ~2010) or could reflect the SRUNet reconstruction having less dynamic range and more noise than real VIIRS. Worth a sensitivity check — this same dataset has both a simulated and a real value for 2012, so comparing those directly would help isolate model artifact from real signal — before treating the 2000→2005 dip as a finding rather than a limitation.

Output: `nightlights_by_grid_year.parquet` (columns: `grid_id`, `year`, `source_type`, `mean_radiance`), exported via `export_nightlights_to_excel.py` with `national_summary` / `by_cell` / `notes` sheets, same convention as Phases 8–10.

### Phase 12 — Urbanisation
GHSL built-up layer. Plan: cross-check against the night-lights series once both exist, since built-up area and night-light intensity should broadly track each other — a large discrepancy would be a useful QA signal for either dataset.

### Phase 13 — Master Spatial Dataset
Merge all variables on `(grid_id, year)`. Given Phase 8's finding, this join needs an explicit decision documented alongside it: roads join as a **static 2024 value** (repeated across all years, or joined only where `year == 2024` depending on the regression design in Phase 14), not as a genuine per-year value like forest/fragmentation/population. `join_tables.py` may already be a first attempt at this — confirm before treating it as final. Recommend resolving the design choice before finalizing the join, not during.

### Phase 14–17 — Statistical Analysis
Regression/correlation between forest loss/fragmentation and development variables. Given the roads caveat, an early step here should be a sensitivity check: run the core regression with and without roads included, to see how much (if at all) conclusions depend on a variable now known to carry a campaign-timing artifact.

### Phase 18 — Lemur Occurrence Data
GBIF records + IUCN range polygons. Worth checking GBIF record density/recency for Madagascar specifically before committing to it as the primary occurrence source — citizen-science-driven databases like GBIF can have their own version of Phase 8's problem (observation effort bias correlating with accessibility/development), so it's worth checking for that pattern proactively rather than discovering it after modelling.

### Phase 19 — Species Distribution Modelling
Standard SDM workflow (e.g. MaxEnt or a random forest/gradient-boosted equivalent) using the Phase 13 master dataset as covariates and Phase 18 occurrences as the response. Given the roads lesson, treat any covariate sourced from crowdsourced/volunteer data with the same scrutiny before trusting it as ecologically meaningful.

### Phase 20 — Habitat Forecasting
Project the Phase 19 model forward using forecasted covariate values.

### Phase 21 — Future Development Scenarios
Define development scenarios (e.g. business-as-usual vs. accelerated infrastructure) to feed into Phase 20's forecast. Note: since roads are being used as a static current-state layer rather than a time-varying predictor (Phase 8/13 decision), a "future roads development scenario" will need a different data source or an explicit assumption-based scenario construction — it cannot be extrapolated from the existing OSM-derived time series, which doesn't represent real infrastructure growth.

### Phase 23 — ML-Based Prediction of Future Species Impact
Combine Phase 20's habitat forecast with Phase 21's scenarios to project impact on selected lemur species.

---

## Known Limitations (Documented, Not Bugs)

- **`gain` layer** only covers 2000–2012; post-2012 regrowth is not captured (JRC TMF considered, deferred).
- **Roads** are only available for 2010, 2015, 2020, 2024 (OSM history starts Oct 2007); the year-over-year trend reflects when mapping campaigns occurred, not real road construction (see Phase 8) — usable as a current-state (2024) layer, not as a time-varying development predictor.
- **Night lights** will require sensor intercalibration (DMSP↔VIIRS) around 2012–2013 to form one continuous series.
- **Canopy threshold** fixed at 25% — a documented, adjustable constant, not a hardcoded assumption.
- **Population 2024** is approximated from WorldPop's 2023 interim mosaic — no true 2024 raster exists (see Phase 9). Population also runs consistently 4–10% below the UN WPP reference across all years — expected for WorldPop's unconstrained product, not a pipeline defect, but worth stating explicitly wherever population is used downstream.
- **WorldPop nodata sentinel** is not uniform across product vintages (the 2000–2020 annual mosaics use a different nodata value than the 2023 interim mosaic) — any future WorldPop pull should read nodata from file metadata rather than assuming a fixed value, per the Phase 9 bug writeup.
- **GLAD Global Cropland Extent** is not annual — only five epochs exist (2003/2007/2011/2015/2019), each labeled by its end year; Phase 10's panel years use the nearest epoch with up to ~3yr of label error (see `source_year`). Confirmed (empirically, not just from docs) to be independent per-epoch snapshots, not cumulative — so it does support measuring real cropland loss, unlike a cumulative product would.
- **ESA WorldCover** only has two real years, 2020 and 2021 — nothing at 2024 (same shape as Phase 9's population gap), and its 2015→2020 splice against GLAD likely contains a methodology-driven level-shift, not a pure trend (see Phase 10 known limitation).
- **GLAD pixel value 0** means "not cropland" OR "no data" — undistinguished in the high-res product, so true data gaps can't be separated from genuine non-cropland within the GLAD-sourced years.
- **Night lights simulated-vs-observed dynamic range** — the pre-2012 "simulated" years (2000/2005/2010) are an order of magnitude smaller and noisier in direction (a slight dip 2000→2005) than the observed years (2015/2020/2024), which show clean, larger growth; not yet confirmed whether the dip is real or a reconstruction-model artifact (see Phase 11 open question).
- **rasterstats `all_touched` default** caused a real bug in Phase 11 (small coastal-sliver cells silently returned no data) — worth checking whether Phases 6/7/9/10 have any equivalent tiny-cell blind spot, even though none showed missing cells in their own QA checks.

---

## Suggested Immediate Next Steps
1. Move the scripts listed above into a proper `scripts/` folder with a consistent import/path convention (currently run from the project root by filename).
2. Delete the superseded `roads_to_excel.py`; `join_tables.py` is confirmed to be an early, incomplete Phase 13 draft — rewrite it once agriculture/nightlights are folded in rather than extending it piecemeal.
3. Decide and document the Phase 13 join design for roads (static 2024 vs. year-conditional) before finalizing the master merge, so it's a deliberate choice rather than an implementation detail discovered later.
4. Consider a same-source sensitivity check for the Phase 10 2015→2020 splice (GLAD 2019 vs. WorldCover 2020, both real adjacent years) to quantify how much of the +87% jump is methodology vs. real change, before it feeds Phase 14–17.
5. Consider the equivalent sensitivity check for Phase 11 (simulated vs. real VIIRS, both available for 2012) to resolve the open question on the flat pre-2010 trend.
6. Begin Phase 12 (urbanisation, GHSL) — the last remaining data-acquisition phase before the Phase 13 master merge.