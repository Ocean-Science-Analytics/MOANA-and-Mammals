# MOANA & Megafauna

**PACE Phytoplankton Communities and Marine Mammal Habitat Use**

*NEFSC Mid-Atlantic Marine Mammal Visual Survey × NASA PACE Satellite Data*

Contributors — Liz Ferguson, Ana Vaz, James King, Sarah Roberts, Katherine Gallagher

## Overview

This repo explores incorporating MOANA and other PACE data products into marine mammal habitat characterization and species distribution modeling. The notebook here integrates NOAA NEFSC cetacean survey observations with NASA PACE satellite data to characterize the environmental niche of marine megafauna in the Mid-Atlantic Bight (Dec 2024 – Mar 2025). Phytoplankton community structure (MOANA: *Synechococcus*, *Prochlorococcus*, picoeukaryotes), chlorophyll-a, particulate organic carbon (PACE OCI), sea surface temperature (MODIS Aqua), and bathymetry (GEBCO) are co-located with individual sighting records, then used to test whether cetacean species differ in the habitat conditions they occupy. The broader aim is to evaluate which PACE-derived variables are most useful as predictors in species distribution models (SDMs) for marine megafauna, alongside more established physical/geographic covariates.

## Data Sources

| Variable | Product | Satellite / Sensor | Resolution |
|---|---|---|---|
| Marine mammal sightings | OBIS-SEAMAP dataset 2368 (NEFSC survey) | — | point observations |
| Chlorophyll-a | `PACE_OCI_L3M_CHL` | PACE / OCI | ~1 km |
| Phytoplankton carbon | `PACE_OCI_L3M_CARBON` | PACE / OCI | ~1 km |
| Synechococcus / Prochlorococcus / Picoeukaryotes | `PACE_OCI_L4M_MOANA` | PACE / OCI | 4 km |
| Sea surface temperature | MODIS Aqua L3 daily | MODIS / Aqua | ~4 km |
| Bathymetry (depth) | GEBCO 2025 grid | — | gridded |
| Sea surface height anomaly | AVISO / CMEMS (`SEALEVEL_GLO_PHY_L4_NRT`) | — | 0.25° |

Satellite products are queried and downloaded via [`earthaccess`](https://earthaccess.readthedocs.io/), NASA's Earthdata access library. GEBCO bathymetry and CMEMS SSHA are not distributed through Earthdata and must be downloaded separately (see **Setup** below).

### Planned: Optical Water Type Classification

MOANA's phytoplankton size-class retrievals are tuned for open-ocean (case 1) waters, so their reliability is expected to degrade in optically complex coastal waters. To evaluate where MOANA can be trusted closer to shore, we plan to incorporate **optical water type (OWT) classification** from hyperspectral PACE reflectance, following the [Fish-PACE Hackweek 2025 water classification tutorial](https://fish-pace.github.io/hackweek-2025/presentations/notebooks/water_classification.html). This approach matches each PACE pixel's hyperspectral R<sub>rs</sub> spectrum (via cosine distance) to one of 23 reference optical water classes (Wei et al. 2022), giving a consistent classification of water type at each observation location. Cross-referencing OWT class against MOANA output will let us flag or filter observations that fall in water types where MOANA's assumptions are less likely to hold, which is particularly relevant for nearshore cetacean sightings.

## Requirements

- Python 3.12
- A free NASA Earthdata account for satellite data access
- (Optional) A Copernicus Marine account if re-extracting SSHA from scratch

Python packages:

```
numpy
pandas
xarray
matplotlib
cartopy
seaborn
earthaccess
tqdm
scipy
scikit-learn
h5netcdf
```

## Setup

1. **Clone the repo and install dependencies:**
   ```bash
   pip install numpy pandas xarray matplotlib cartopy seaborn earthaccess tqdm scipy scikit-learn h5netcdf
   ```

2. **Authenticate with NASA Earthdata.** The first run of the notebook prompts an interactive login and caches credentials in `~/.netrc`. You'll need a free account at [urs.earthdata.nasa.gov](https://urs.earthdata.nasa.gov/).

3. **Get the survey observations.** Download the NEFSC Mid-Atlantic marine mammal visual survey CSV from OBIS-SEAMAP ([dataset 2368](https://seamap.env.duke.edu/dataset/2368), DOI `10.82144/4431190b`) and place it at:
   ```
   data/OBIS_NEFSC_offshore_obs.csv
   ```

4. **Get GEBCO bathymetry.** Download a NetCDF grid cropped to the study area (35.5–41.0°N, −77.0 to −71.5°E) from [download.gebco.net](https://download.gebco.net) and save it as:
   ```
   data/gebco_study_area.nc
   ```

5. **(Optional) Get SSHA.** If you want sea surface height anomaly re-extracted from scratch rather than already present in your CSV, register at [marine.copernicus.eu](https://marine.copernicus.eu), download daily gridded SSHA (`SEALEVEL_GLO_PHY_L4_NRT`) for the study period, and point `SSHA_DIR` to the folder of NetCDF files in Section 3g.

## Notebook Structure

1. **Setup & Authentication** — imports, plot styling, study-area bounding box, species colors, figure export helper (TIFF, 300 dpi, for RSE submission), and Earthdata login.
2. **Load Survey Observations** — reads and parses the NEFSC sighting CSV.
3. **Satellite Data Extraction** — for each observation, queries the relevant satellite product and extracts summary statistics (mean, median, std, min, max, Q25, Q75) from a 3×3 pixel window centered on the sighting:
   - 3a. Shared pixel-window extraction helper
   - 3b. PACE OCI chlorophyll-a
   - 3c. PACE OCI phytoplankton carbon
   - 3d. PACE MOANA phytoplankton community composition
   - 3e. MODIS Aqua sea surface temperature
   - 3f. GEBCO bathymetry / depth
   - 3g. Sea surface height anomaly (optional, external source)
   - 3h. Merges and saves the combined dataset to `data/NEFSC_All_Variables.csv`

   > If `data/NEFSC_All_Variables.csv` already exists, you can skip Section 3 and start at Section 4.

4. **Study Area Maps** — bathymetry map with observation overlays, and MOANA phytoplankton community maps for two representative survey dates.
5. **Environmental Conditions by Species** — box plots of chlorophyll-a, depth, SST, phytoplankton carbon, and the three MOANA variables, stratified by species (species with n ≥ 20 observations: fin, humpback, sperm, and common minke whales, plus beaked whales).
6. **Environmental Niche Analysis** — summary statistics and Kruskal-Wallis tests across species; pairwise Mann-Whitney U tests with Bonferroni correction; Pianka's niche overlap index for each variable × species pair; comparison of biological vs. physical/geographic variable overlap.
7. **Multivariate Analysis (PCA)** — reduces the environmental variable space to principal components to visualize species separation and identify which satellite-derived variables drive it.

## Outputs

- `data/NEFSC_All_Variables.csv` — merged sightings + satellite variables, used for all downstream analysis.
- `figures/*.tiff` — all figures, exported at 300 dpi per Remote Sensing of Environment (RSE) submission requirements.

## Notes

- The MODIS SST `short_name` is current as of early 2026; if granules aren't found, verify the identifier at [search.earthdata.nasa.gov](https://search.earthdata.nasa.gov).
- Re-running Section 3 overwrites `data/NEFSC_All_Variables.csv`.
- Optical water type classification (see above) is not yet implemented in this notebook; it's a planned addition for assessing MOANA reliability in nearshore waters.
