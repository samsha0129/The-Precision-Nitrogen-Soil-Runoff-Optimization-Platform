# 🌱 Precision Nitrogen & Soil Runoff Optimization Platform

A geospatial ML platform that turns open satellite imagery, soil data, and weather
forecasts into **zone-level variable-rate nitrogen prescriptions** for agricultural
fields — maximizing crop yield while keeping the probability of nitrogen leaching or
runoff below a configurable risk threshold.

Flat-rate fertilizer application ignores the fact that a single field can contain
sandy, fast-draining patches that leach nitrate straight into groundwater right next
to loam that retains it efficiently. This project automates the alternative:
satellite-derived vegetation/moisture indices + soil texture are clustered into
management zones, and a physics-informed ML model recommends a different nitrogen
rate per zone under the upcoming 5-day forecast.

## Key Features

- 🛰️ **Satellite ingestion** — pulls free Sentinel-2 L2A surface reflectance (Earth
  Search STAC API) and computes NDVI, NDWI, and SAVI with cloud/shadow masking via
  the Scene Classification Layer
- 🧪 **Soil fusion** — combines spectral indices with ISRIC SoilGrids texture/carbon
  data, deriving water retention and drainage rate via Saxton & Rawls (2006)
  pedotransfer functions (SoilGrids doesn't publish hydraulics directly)
- 🗺️ **Automated zoning** — PCA + silhouette-optimized K-Means (k=3–5) segments a
  field into agronomically labeled management zones (e.g. *High-Leaching Sand*,
  *Saturated Lowland*), with a rule-based override so small-but-critical wet spots
  aren't averaged away
- ⚖️ **Dual-objective optimization** — a Monte Carlo soil-water/nitrate simulator
  generates training data for a monotone-constrained, dual-output XGBoost surrogate
  (yield response + leaching risk), so every recommendation is fast *and* physically
  consistent (more N or rain can never predict *less* risk)
- 📊 **Interactive advisor portal** — draw a field boundary on a satellite map,
  review color-coded zones and a variable-rate map, and export a prescription as a
  Shapefile for real tractor rate controllers
- ✅ **Benchmarked against a flat rate** — every run reports applied N, N lost, and
  yield gain versus a single field-average rate, so the value of zoning is a number,
  not an assertion

## Tech Stack

| Layer | Tools |
|---|---|
| Geospatial processing | Python 3.10+, GeoPandas, Rasterio, Shapely |
| Machine learning | Scikit-learn (K-Means, PCA), XGBoost (monotone-constrained dual-output regressor) |
| Simulation | NumPy / SciPy (Monte Carlo soil water & nitrate transport model) |
| Web app | Streamlit, Folium / streamlit-folium |
| Data sources | Sentinel-2 L2A (Earth Search STAC, AWS Open Data), ISRIC SoilGrids 2.0, Open-Meteo forecast API |

## Getting Started

### Prerequisites
- Python 3.10+
- `pip`

### Installation
```bash
git clone https://github.com/<your-username>/precision-nitrogen-platform.git
cd precision-nitrogen-platform
pip install -r requirements.txt
```

No API keys or environment variables are required — Earth Search, SoilGrids, and
Open-Meteo are all public, unauthenticated endpoints.

### Verify the install
```bash
python spectral_engine.py selftest
python clustering.py selftest
python nitrogen_model.py selftest
```

## Usage

### Run the interactive app
```bash
streamlit run app.py
```
Choose **Synthetic demo** to try it fully offline (no network calls), or **Live
data** to run against real Sentinel-2 imagery and SoilGrids for a field you draw or
upload. The first live run trains the nitrogen model (~1–2 minutes); it's cached for
subsequent runs.

### Command line (no UI)
```bash
# Phase 2: fetch imagery + soil data and generate management zones
python clustering.py real \
  --bbox -121.535 36.560 -121.515 36.575 \
  --start 2025-06-01 --end 2025-07-31 \
  --out out/field1 --cache cache/soilgrids

# Phase 3: train the nitrogen model and generate a prescription
python nitrogen_model.py train --crop corn --out models/corn
python nitrogen_model.py recommend \
  --model models/corn \
  --zones out/field1/zone_profile.csv \
  --zones-geojson out/field1/zones.geojson \
  --stage V8-V10 --prior-n 40 \
  --out out/field1/rx.csv
```

### Output
- `*_shp.zip` — zone polygons with rate attributes, importable by farm rate
  controllers / FMIS software
- `*_grid.csv` — point-grid prescription (lat, lon, N rate)
- `*_report.json` — full audit trail: imagery scene ID, soil source, forecast,
  model metrics, and warnings

> ⚠️ Advisory tool. Verify rates with local agronomic guidance and in-season
> soil/tissue tests; follow label and regulatory limits.
