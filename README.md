# Kerala Precipitation Analysis using GPM IMERG via Google Earth Engine (XEE)

A Python workflow that pulls daily precipitation data directly from Google Earth Engine into `xarray` using [`xee`](https://github.com/google/Xee), clips it to Kerala, and analyzes daily/annual rainfall — without needing to manually export rasters from GEE first.

## Overview

This notebook demonstrates an end-to-end pipeline for working with GEE-hosted precipitation data via the `xee` Xarray backend:

1. **Authenticate & initialize** Earth Engine (via the high-volume endpoint, suited for bulk pixel reads)
2. **Load Kerala's boundary** from a GADM administrative shapefile and derive a bounding box
3. **Build a daily precipitation `ImageCollection`** from NASA's GPM IMERG V07 by summing half-hourly precipitation into daily totals over the desired date range
4. **Open the collection lazily as an `xarray.Dataset`** using `xee`'s `engine='ee'` backend
5. **Fix `xee`'s non-standard dimension order and CRS handling** so the data plays nicely with `rioxarray`
6. **Clip** the grid to Kerala and compute spatial-average daily/annual rainfall
7. **Save to NetCDF** locally with compression, and validate by reloading

## Data Sources

| Data | Source |
|---|---|
| Daily precipitation | [NASA/GPM_L3/IMERG_V07](https://developers.google.com/earth-engine/datasets/catalog/NASA_GPM_L3_IMERG_V07) via Google Earth Engine, accessed through `xee` |
| Kerala state boundary | [GADM v4.1](https://geodata.ucdavis.edu/gadm/gadm4.1/shp/gadm41_IND_shp.zip) |

## Method Notes — things that aren't obvious from the GEE/xee docs

- **`ee.Initialize()` requires a Google Cloud Project ID** — pass `project='your-gcp-project-id'` or initialization fails.
- **`scale` in `xee` is in the units of your CRS**, not always meters. With `EPSG:4326` (used here), `scale=0.1` means 0.1° pixels — matching IMERG's native ~10 km resolution. In a projected CRS (e.g. `EPSG:3857`), the same parameter would mean meters instead.
- **`xee` returns data with dimension order `(time, lon, lat)`**, not the `(time, lat, lon)` convention used by CF-compliant NetCDF and most other geospatial Python tooling. This notebook explicitly transposes to `('time', 'lat', 'lon')` right after opening the dataset to avoid silent axis-order bugs downstream.
- **`rioxarray` expects `x`/`y` dimension names by default.** Since this pipeline keeps `lat`/`lon` naming (for consistency with other IMD-based work), spatial dims are explicitly declared via `ds.rio.set_spatial_dims(x_dim='lon', y_dim='lat')`, and the CRS is written explicitly via `ds.rio.write_crs('EPSG:4326')` — the `crs` attribute that `xee` sets on the dataset is just metadata text and isn't enough on its own for `rioxarray` to recognize it.
- Clipping renames to `x`/`y` at the point where `rioxarray`'s `.rio.clip()` is called, since that's where the rename is actually required.

## Tech Stack

- [`earthengine-api`](https://developers.google.com/earth-engine) (`ee`) — Earth Engine access
- [`xee`](https://github.com/google/Xee) — Xarray backend for Earth Engine
- `xarray` — gridded/labelled array handling
- `rioxarray` — CRS, spatial dimensions, and clipping
- `geopandas` / `shapely` — vector boundary handling
- `matplotlib` — visualization

## Repository Structure

```
.
├── XEE_IMERG_precip.ipynb    # Main analysis notebook
└── README.md
```

## Outputs

- Kerala-clipped precipitation overlay maps
- Daily spatially-averaged precipitation time series
- Total annual rainfall trend for Kerala
- Local NetCDF export (`kerala_gpm_imerg_daily_*.nc`), reloaded and validated

## Getting Started

```bash
pip install earthengine-api xee xarray rioxarray geopandas shapely matplotlib
```

You'll need:
1. A Google account [registered for Earth Engine access](https://code.earthengine.google.com/register)
2. A Google Cloud Project with the Earth Engine API enabled

Then open `XEE_IMERG_precip.ipynb`, run `ee.Authenticate()` once (credentials are cached locally afterward), and run all cells.

**Note:** pulling multi-year daily data directly through `xee` can be slow and may hit Earth Engine request limits — for long time ranges, consider chunking the pull by year and merging afterward with `xr.open_mfdataset()`.

## Possible Extensions

- Cross-validate against the parallel IMD gridded-rainfall analysis for the same region/period
- Extend to seasonal (JJAS monsoon) and Mann-Kendall trend analysis, as done for the IMD dataset
- Pull at native 0.1° resolution and compare against IMD's 0.25° grid to assess downscaling differences
- Automate yearly chunked export for longer time ranges

## License

Add a license of your choice (e.g., MIT) here.
