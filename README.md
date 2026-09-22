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
│   │   └── gain/            — regrowth 2000–2012, 4 raw tiles + processed
│   ├── raw/                 — reserved for population, agriculture, night lights source rasters
│   └── processed/
│       ├── forest_masks/                     — binary forest/non-forest rasters, one per year
│       ├── madagascar_boundary_projected.gpkg
│       ├── madagascar_grid_10km.gpkg
│       ├── madagascar_grid_clipped.gpkg      — spatial unit of analysis (7,036 cells; id/grid_id field)
│       └── statistics/
│           ├── forest_by_grid_year.parquet               — grid_id, year, forest_pct, forest_pixels
│           ├── fragmentation_by_grid_year.parquet        — grid_id, year, patches, edge
│           ├── master_forest_fragmentation.parquet       — forest + fragmentation merged
│           ├── roads_by_grid_year.parquet                — grid_id, year, road_length_m, road_density_km_per_km2 (2010/2015/2020/2024)
│           ├── roads_by_grid_year.xlsx                   — Excel export for viewing
│           ├── roads_mapping_effort_by_grid_year.parquet — diagnostic only: highway_contributor_count per cell per interval
│           └── road_growth_hotspots.gpkg                 — QGIS layer: road growth by cell/period, for visual QA
├── qgis/                      — QGIS project files, visual QA only
├── packages/, venv/           — environment
└── madagascar_habitat.qgz     — QGIS project (visualization/inspection, not processing)
```

### Scripts produced so far (not yet organized into a `scripts/` folder — recommended next housekeeping step)

| Script | Purpose |
|---|---|
| `phase8_roads_ohsome.py` | Pulls time-varying road length + highway-contributor-count from the ohsome API |
| `export_roads_final_to_excel.py` | Exports `roads_by_grid_year.parquet` to Excel with national summary + usage notes |
| `check_road_growth_concentration.py` | Diagnoses whether road-length growth is spatially concentrated (import/campaign signature) |
| `check_effort_vs_growth_concentration.py` | Compares road-growth concentration against contributor-activity concentration, period by period |
| `export_growth_hotspots_qgis.py` | Exports per-cell road growth as a GeoPackage layer for QGIS choropleth mapping |
| `investigate_hotspot_changesets.py` | Pulls raw OSM changeset/contribution metadata for specific hotspot cells via ohsome |
| `check_changeset_metadata.py` | Pulls real changeset metadata (user, comment, editor, hashtags) from OSM's own API |
| `phase9_population_worldpop.py` | Pulls WorldPop annual population rasters and zonal-sums onto the grid (drafted, not yet confirmed run) |

---

## Pipeline: Completed Work

### Phase 1–2 — Project Setup, Boundary & Grid
Defined the Madagascar boundary and built a 10km fishnet grid, clipped to the coastline (`madagascar_grid_clipped.gpkg`, 7,036 cells). This is the fixed spatial unit every later variable is aggregated into.

### Phase 3 — Forest Data Acquisition
Downloaded Hansen Global Forest Change v1.12 tiles (4 tiles covering Madagascar) for three variables: `treecover2000` (2000 baseline % canopy cover), `lossyear` (year of loss, 2001–2024), `gain` (binary regrowth, 2000–2012 only, no per-year attribution).

### Phase 4 — Preprocessing
Mosaicked each variable's 4 tiles, clipped to the Madagascar boundary, reprojected to EPSG:3857 to match the grid.
**Bug resolved:** an initial clip silently failed (output raster was the full unclipped tile rectangle, including Comoros/Mauritius) — fixed by force-clipping the final output array rather than trusting upstream inputs.

### Phase 5–6 — Forest Area & Change (Derivation)
Hansen doesn't provide a direct per-year forest layer, so one was derived: a pixel counts as forest in year Y if it was ≥25% canopy in 2000 AND hasn't yet appeared in `lossyear` by year Y, with `gain` pixels added back in from 2013 onward (documented simplification — `gain` has no per-year timestamp within its 2000–2012 window). Produced `forest_masks/forest_{year}.tif` for 2000, 2005, 2010, 2015, 2020, 2024.

**Performance issue resolved:** initial rasters were untiled/uncompressed (1-pixel-tall row strips), making zonal stats impractically slow — fixed via tiled, ZSTD/deflate-compressed GeoTIFF writes.
**Considered, deferred:** JRC TMF as a supplementary dataset to cover regrowth beyond 2012 — marginal gain judged not worth the added complexity.

**Zonal statistics:** % forest cover per grid cell per year computed via rasterstats/exactextract → `forest_by_grid_year.parquet`. National trend verified as a smooth, plausible decline (~30.7% in 2000 → ~22.2% in 2024).

### Phase 7 — Fragmentation
Patch count and total edge length per grid cell per year, computed via `pylandstats` directly on `forest_masks/forest_{year}.tif` (no new external data).
**Bugs resolved:** (1) wrong method name (`class_metrics` → `compute_class_metrics_df`); (2) a nodata/class-collision bug where pylandstats' default treatment of 0 as "nodata" silently zeroed out all edge-length calculations while leaving patch counts unaffected.
Output: `fragmentation_by_grid_year.parquet`.

### Phase 8 — Roads ✅ Complete
Originally scoped as a static layer (assumption: "OSM has no historical archive"). This assumption was revisited and found to be only partially true.

**Method:** the ohsome API (HeiGIT) exposes OSM's full edit history and allows querying road length as of a past date. Pulled per grid cell for 2010, 2015, 2020, 2024 via `/elements/length/groupBy/boundary`.
**Hard limit found:** 2000 and 2005 are unavailable — OSM's tracked history only starts 2007-10-08, and ohsome rejects a multi-timestamp request entirely if any timestamp predates that (this caused an initial batch of confusing identical 404s across all cells).

**Anomaly investigated:** national road length grew sharply and non-monotonically (6,542 km → 57,756 km → 127,934 km → 494,818 km across 2010/2015/2020/2024). Investigation process:
1. **Concentration check** — growth is spatially concentrated in a small share of cells each period (e.g. 50% of 2015→2020 growth in just 4.2% of cells), increasingly so over time.
2. **Mapping-effort covariate** (unique contributors, all edit types) didn't track the growth pattern — ruled out a simple "more people mapping" explanation.
3. **Direct changeset inspection** (ohsome `/contributions/geometry`, then OSM's own Changesets API) revealed the true cause: manual JOSM edits by named volunteer mappers, organized under specific project hashtags — a geo-health research project (`#geohealthresearch-project-14`, tied to Institut Pasteur Madagascar / IRD / Pivot Madagascar) and Operation Fistula's Missing-Maps-style campaign (`#opfistula #projectfree`), concentrated around Andramasina, Analamanga, mid-2022.

**Conclusion:** not a bulk/bot import, but a **mapping-campaign-timing artifact** — health/humanitarian NGOs mapped these areas because they needed maps for service delivery, which has no necessary relationship to (and may anti-correlate with) local development level.

**Decision:** `roads_by_grid_year.parquet` is kept in full as the Phase 8 deliverable, but only the **2024 snapshot** will be joined into modelling — as a current-state road-network layer for Phase 18–21, not as a time-varying predictor in the Phase 14–17 regression.

**Bug resolved and confirmed (post-completion):** `road_density_km_per_km2` came back all-`NaN` due to a type mismatch — the area lookup dict was keyed by `grid_id` in its native (integer) dtype from the grid file, while ohsome returns `grid_id` as a string, so every dict/`.map()` lookup silently failed. Fixed by casting `grid_id` to string immediately after loading the grid, in every script that touches it. Re-run confirmed sane values: national density rose from 0.0098 km/km² (2010) to 0.7343 km/km² (2024), consistent with total length ÷ Madagascar's ~587,000 km² landmass.

Also produced `roads_mapping_effort_by_grid_year.parquet` (highway-specific contributor counts per interval) as a diagnostic/QA artifact — not part of the master table.

---

## Pipeline: Remaining Work

### Phase 9 — Population *(drafted, not yet confirmed run)*
**Resource:** WorldPop unconstrained global population mosaics, 1km resolution, genuine annual coverage 2000–2020 (`https://data.worldpop.org/GIS/Population/Global_2000_2020/{year}/0_Mosaicked/ppp_{year}_1km_Aggregated.tif`). Unlike roads, this is a real per-year time series, not subject to a mapping-effort artifact.
**Known gap:** no true 2024 raster exists; the script substitutes WorldPop's 2023 "interim" global mosaic, labeled `source_year=2023` so the substitution is traceable downstream.
**Method:** zonal **sum** (not mean — population is a count, like forest pixel counts) per grid cell, streamed via GDAL's `/vsicurl/` to avoid downloading full ~1.1GB global rasters.
**Next action:** run `phase9_population_worldpop.py`, sanity-check grid totals against the UN population reference figures printed by the script (national total should land within a reasonable range of the reference — large gaps indicate a CRS/nodata/clip issue, same QA logic as the Phase 6 forest sanity check).

### Phase 10 — Agriculture
GLAD Global Cropland (2000–2019) + ESA WorldCover (recent years) for % cropland per cell. Two source datasets will need harmonizing at their boundary year(s); worth deciding the splice year up front rather than discovering a discontinuity later (same class of issue as Phase 11's sensor intercalibration).

### Phase 11 — Night Lights
DMSP-OLS (pre-2013) + VIIRS DNB (2012–present). These two sensors are not directly comparable — intercalibration is required to form one continuous series across the ~2012–2013 splice point. This is a well-documented remote-sensing problem with established calibration approaches in the literature; worth a literature check before implementing rather than deriving calibration coefficients from scratch.

### Phase 12 — Urbanisation
GHSL built-up layer. Plan: cross-check against the night-lights series once both exist, since built-up area and night-light intensity should broadly track each other — a large discrepancy would be a useful QA signal for either dataset.

### Phase 13 — Master Spatial Dataset
Merge all variables on `(grid_id, year)`. Given Phase 8's finding, this join needs an explicit decision documented alongside it: roads join as a **static 2024 value** (repeated across all years, or joined only where `year == 2024` depending on the regression design in Phase 14), not as a genuine per-year value like forest/fragmentation/population. Recommend resolving this design choice before writing the join, not during.

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
- **Population 2024** will be approximated from WorldPop's 2023 interim mosaic — no true 2024 raster exists yet at time of writing.

---

## Suggested Immediate Next Steps
1. Run `phase9_population_worldpop.py` and confirm output against the printed UN reference sanity check.
2. Move the scripts listed above into a proper `scripts/` folder with a consistent import/path convention (currently run from the project root by filename).
3. Decide and document the Phase 13 join design for roads (static 2024 vs. year-conditional) before writing the master merge, so it's a deliberate choice rather than an implementation detail discovered later.