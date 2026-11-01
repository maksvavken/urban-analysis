# Changes log: what we changed and what it resulted in

*Covers 30 Oct 2026: notebooks `04_poi`, `05_streets`, `06_baseline_clustering`, `07_radius_counts`.*
*Earlier work (01–03) is explained in `progress_explained.md`.*
*(Slovene version: `changes_log_sl.md`.)*

Each entry: **change → why → result**. Numbers are from the runs on 30 Oct 2026.

---

## A. Decisions about which data to use

### A1. Urban Atlas dropped, OSM POI used for "function" instead
- **Why:** Urban Atlas classes are predefined land-use categories. The research question asks for types that emerge *without predefined labels*, and clusters built on Urban Atlas could just reproduce its classes.
  (Note: using it would not make the method "supervised"; class shares would only be input features. The argument is about not baking in someone else's categories.)
- **Result:** function is described by counts of points of interest (04_poi). The supervisor's original list had Urban Atlas, so this needs to be shown to Špela.

### A2. Household size dropped
- **Why:** not in the SURS file we have; not essential for the baseline.
- **Result:** the "people" layer has one clustering indicator (population density). Age shares (0–14, 65+) are used only for profiling, never as input (circularity rule).

### A3. PT accessibility left out of the baseline
- **Why:** it overlaps with population, POI and street density, and is mostly a centre-to-edge gradient that could dominate the clusters. Following "one layer at a time", it can be added later as its own layer and its effect measured (ablation).
- **Result:** no GTFS download needed for now.

### A4. Relief (DEM) postponed
- **Why:** topography is only a control in the LST evaluation, never an input.
- **Result:** not needed until the LST step; a 30 m DEM (Copernicus GLO-30) is enough for a 100 m grid.

---

## B. Getting the OSM data

### B1. Online download (osmnx / Overpass API) → failed
- **What happened:** first the server rejected the MOL boundary as a query polygon (413, too many vertices). After simplifying it, the main server and its mirror were overloaded (502 / timeouts).
- **Result:** abandoned; nothing was saved.

### B2. Manual download from Geofabrik (Slovenia shapefiles, 30.10.2026)
- **Why:** one file, no server queries, reproducible (date recorded in `data/raw/OSM/PREBERI.txt`).
- **Result, with side effects that shaped the code:**
  - POI already come with a type (`fclass`), so categories are a simple mapping.
  - The free extract has **no offices** → workplaces are under-represented in `poi_services`.
  - Roads have no `service` subtype and no sidewalk tag → parking aisles can't be separated from access roads, and sidewalks can't be separated from footpaths (see C2).

---

## C. Building the indicators

### C1. POI categories (04_poi)
- **Change:** 6 categories (food, retail, education, health, services, leisure). Dropped: street furniture (benches, bins, cameras), monuments/artworks, parks, pitches and playgrounds (greenery is its own layer; hundreds of pitches in fields), car-sharing/bike-rental stations (mobility is not in the baseline).
- **Result:** 3,181 of 10,988 OSM objects kept; 3,172 inside MOL. Top cells: old town and shopping centres (Šiška, Rudnik).
  **Problem found:** only 1,523 of 27,505 cells (5.5 %) have any POI. → led to D5.

### C2. Street network (05_streets)
- **Change 1:** forest/field tracks excluded → **result:** forests don't look like they have streets.
- **Change 2:** `service` roads excluded → **result:** no thousands of fake intersections/dead ends in car parks. Side effect: a street that continues as a service road counts as a dead end.
- **Change 3:** footpaths/cycleways not used for intersections, only for `str_path_share` → **result:** zebra crossings and separately drawn sidewalks don't create fake intersections in the centre.
- **Change 4:** intersection nodes within 10 m merged → **result:** 5,033 nodes → 4,677 intersections (one junction counts once). 1,952 dead ends.

### C3. Bug fix: intersections lost
- **What happened:** after merging nearby nodes, all intersections had the same index, and the "drop duplicate matches" step removed almost all of them (max 1 intersection per cell).
- **Fix:** reset the index.
- **Result:** max 13 intersections per cell; the top cell is the old town (median segment 24 m), as expected.

### C4. Jupyter kernel crash
- **What happened:** the `python3` kernel calls plain `python`, which without an activated env is the system Python → kernel died.
- **Fix:** run Jupyter / nbconvert with the `urban` env activated.

---

## D. Baseline clustering

### D1. Choosing 6 indicators, one per layer
- **Why:** with all ~26 indicators, layers with many columns (POI 8, buildings 6) would dominate layers with one (population).
  Standardisation balances *indicators*, not *layers*.
- **How:** Spearman correlations among the 8,207 cells with buildings (over all cells the many empty cells inflate every correlation).

| layer | chosen | not chosen, because |
|---|---|---|
| people | `pop` | – |
| ground surface | `imp_share` | – |
| buildings | `bld_height` | coverage and FAR repeat imperviousness (ρ 0.81 / 0.80); height doesn't (0.59) |
| greenery | `can_cover` | NDVI is almost the mirror of imperviousness (ρ −0.91); canopy is nearly independent (−0.30) and separates trees from grass/fields |
| function | `poi_count` | diversity is empty in 94 % of cells |
| streets | `str_intersections` | street length overlaps with it (0.68) and correlates more with the other layers |

- **Result:** no pair among the chosen 6 has |ρ| above about 0.6.

### D2. Which cells: option A (building data extent only)
- **Why:** EUBUCCO covers the settlement of Ljubljana, not all of MOL; outside it, building height is unknown (not zero).
- **Result:** 16,369 cells − 18 water cells (`ndvi_neg_share > 0.5`) = **16,351 cells** with 290,960 of 300,443 residents.
  885 populated edge cells (9,481 residents) are left out → ask Špela for the MOL extract.

### D3. Transformations
- **Dorm cell capped** at the second-highest population → it stays "very dense" without forming its own cluster.
- **log(1 + x)** for population, POI, intersections → the few extreme cells no longer squeeze all ordinary cells together.
- **z-score** for all 6 → equal weight in the k-means distance (otherwise the indicator with the largest numbers decides).

### D4. Choosing k (06)

| k | silhouette | Davies–Bouldin | seed ARI mean (min) | smallest cluster |
|---|---|---|---|---|
| 4 | 0.44 | 1.10 | 0.68 (0.42) | 1,379 |
| 5 | 0.47 | 0.98 | 0.77 (0.41) | 1,270 |
| **6** | **0.49** | **0.95** | **0.95 (0.80)** | 710 |
| 7 | 0.50 | 1.05 | 0.89 (0.80) | 398 |
| 9 | 0.51 | 1.06 | 0.87 (0.76) | 171 |

- **Result:** **k = 6**: best seed stability and best Davies–Bouldin; silhouette only creeps up afterwards while clusters get small.

### D5. Baseline result (06, k = 6)
| cluster | reading |
|---|---|
| 0 | forest, wooded hills |
| 1 | open land: fields, meadows, Barje |
| 2 | residential, few intersections in the cell |
| 3 | residential, intersections in the cell |
| 4 | non-residential built-up: industry, BTC, logistics |
| 5 | centre and mixed-use |

- **Problem found:** clusters 2 and 3 differ almost only by whether an intersection happens to lie inside the 100 m cell (mostly 0 or 1) → "salt and pepper" pattern in residential areas.

---

## E. Radius counts (07)

### E1. POI and intersections counted within 300 m instead of inside the cell
- **Change:** for each cell, the POI and intersections in all cells whose centre is within 300 m (29 cells), divided by the number of grid cells in that window → per hectare in the neighbourhood. Same pipeline otherwise, same k = 6.
- **Result:**

| | baseline (in cell) | radius 300 m |
|---|---|---|
| cells with any POI | 1,523 | 9,352 |
| cells with any intersection | 2,921 | 14,924 |
| neighbour agreement (speckle; higher = more coherent) | 0.63 | **0.71** |
| ARI vs baseline | – | 0.81 |
| silhouette, k = 6 | 0.49 | 0.42 (not comparable, see below) |
| seed ARI, k = 6 | 0.95 | 0.86 |
| most seed-stable k | 6 | **5** (ARI 0.997) |

- **What changed in the clusters:**
  - Forest (99 % of cells stay), open land (94 %) and non-residential (88 %) change little.
  - The residential split now means something: **low-density residential** (median 20 residents/ha, 7.6 m height) vs **dense urban residential** (65 residents/ha, 9.5 m, shops and intersections nearby; 182,000 residents). The old split by chance intersections mixes into both.
  - The centre cluster shrinks from 710 to 458 cells and becomes compact (old town + a few secondary centres).
- **Caveats:**
  - Silhouette is not comparable across feature sets: the in-cell counts were mostly exact zeros, which make clusters look artificially tight.
  - k = 5 merges the two residential clusters. If the radius version becomes the baseline, k must be chosen again (stability vs a meaningful split).

---

## F. Radius sensitivity (08)

### F1. Same pipeline with 200 m and 500 m
- **Change:** none to the data; 07 repeated with radius 200 m (13-cell window) and 500 m (81 cells), plus the in-cell version as reference. Output: `radius_sensitivity.csv`, `08_scores_by_radius.png`, `08_maps_by_radius.png`.
- **Result:**

| | in cell | 200 m | 300 m | 500 m |
|---|---|---|---|---|
| neighbour agreement, k = 5 | 0.63 | 0.74 | 0.75 | 0.75 |
| neighbour agreement, k = 6 | 0.63 | 0.68 | 0.71 | 0.73 |
| seed ARI, k = 5 | 0.77 | **0.999** | **0.997** | 0.89 |
| seed ARI, k = 6 | 0.95 | 0.92 | 0.86 | 0.89 |
| most seed-stable k | 6 | 5 | 5 | 8 |
| ARI vs 300 m, k = 5 | 0.75 | 0.93 | – | 0.92 |
| ARI vs 300 m, k = 6 | 0.81 | 0.81 | – | 0.91 |

- **What it means:**
  - Any radius removes most of the speckle; beyond 300 m the gain is small (0.71 → 0.73 at k = 6, none at k = 5).
  - At k = 5 the radius hardly matters (ARI 0.88–0.93 between all radii).
  - At k = 6, 300 m and 500 m give the same structure: low-density residential (20–23 residents/ha) vs dense urban residential (65–72 residents/ha). The centre cluster shrinks with the radius (458 → 270 cells).
  - At 200 m the k = 6 split is different: a small sparse fringe cluster (7 residents/ha, 26,000 residents) vs one big residential cluster (197,000), and a larger centre (617 cells). 200 m is too small a neighbourhood to separate the two residential types.
- **Decision (proposed):** keep **300 m** as the radius: the results agree with 500 m, k = 5 is very stable, and the centre stays compact without being smoothed away. k (5 vs 6) is still open.

---

## Open questions
1. Make the radius version the new baseline? 200/500 m checked (F1): 300 m is robust. Still to decide: k = 5 (most stable) vs k = 6 (residential split).
2. For Špela: Urban Atlas → POI swap, indicator choice, k = 6 map; ask for the EUBUCCO MOL extract and the UHI layer.
