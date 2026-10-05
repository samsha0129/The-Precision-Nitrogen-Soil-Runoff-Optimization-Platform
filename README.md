# 🌱 Precision Nitrogen & Soil Runoff Optimization Platform

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Status](https://img.shields.io/badge/status-research%20prototype-orange)

An end-to-end geospatial machine learning system that turns free satellite imagery,
soil data, and weather forecasts into zone-level variable-rate nitrogen prescriptions,
maximizing crop response while keeping the probability of nitrogen leaching or runoff
below a set risk threshold.

Most fields still receive a single flat rate of nitrogen fertilizer, even though one
field can contain sandy, fast-draining soil that sends nitrate straight to groundwater,
waterlogged low spots that lose it to runoff and denitrification, and well-drained loam
that actually uses it. The result is wasted fertilizer and contaminated aquifers, a
documented problem in regions like California's Salinas Valley. This project treats
nitrogen application as a constrained optimization problem: split the field into
management zones from what the soil and canopy actually look like, then choose the
dose for each zone that captures the most yield without exceeding a leaching-risk
limit under the upcoming 5-day forecast.

The system is built in four layers, each a standalone, runnable module:

1. **`spectral_engine.py`** — Sentinel-2 ingestion and spectral index engine
2. **`clustering.py`** — soil fusion and K-Means management-zone segmentation
3. **`nitrogen_model.py`** — Monte Carlo simulator and dual-objective XGBoost recommender
4. **`app.py`** — Streamlit / Folium advisor portal with prescription export

---

## Key Features

- **Real, open data sources, no API keys.** Sentinel-2 L2A surface reflectance comes
  from the Earth Search STAC API (AWS Open Data), soil properties from ISRIC
  SoilGrids 2.0, and the 5-day precipitation and evapotranspiration forecast from
  Open-Meteo. The satellite reader fetches only the byte ranges covering the field,
  re-scores each candidate scene by its own cloud mask over the field, and handles
  the Sentinel-2 processing-baseline reflectance offset that silently breaks many
  pipelines.
- **Agronomic spectral indices with guardrails.** NDVI, NDWI (Gao's canopy-moisture
  form), and SAVI are computed per 10 m pixel after cloud and shadow masking. The
  index functions reject raw digital numbers, since SAVI's soil term is only valid
  on 0–1 reflectance.
- **Format-faithful synthetic data.** A synthetic field generator writes genuine
  Sentinel-2-format granules (uint16 DN with offset, 10 m and 20 m bands, scene
  classification layer) containing a sandy paleochannel and a waterlogged swale, so
  the offline demo runs through the exact same ingestion code as live data and
  ships with ground-truth rasters for validating the zoning.
- **Soil hydraulics derived, not assumed.** SoilGrids publishes texture and carbon
  but not water retention or drainage, so the Saxton & Rawls (2006) pedotransfer
  functions convert sand, clay, and organic matter into wilting point, field
  capacity, available water, and saturated conductivity. Drainage class is estimated
  from conductivity and imagery-detected excess moisture.
- **Agronomically labeled management zones.** PCA plus silhouette-optimized K-Means
  (k = 3–5, preferring fewer zones when scores are close) segments each field, then a
  minimum-patch sieve keeps zones large enough for a spreader to follow. Zones are
  named from soil physics, such as *High-Leaching Sand* and *Saturated Lowland*. A
  rule-based override isolates small waterlogged spots that K-Means would otherwise
  absorb into neighboring loam.
- **A dual-objective nitrogen model.** No public dataset pairs nitrogen dose with
  both yield and measured leaching under a known forecast, so a transparent process
  simulator (daily root-zone water balance, nitrate displacement, runoff,
  denitrification, quadratic-plateau crop response) generates training data across
  128 randomized forecast scenarios per case. Two XGBoost heads learn yield gain and
  leaching risk under **monotone constraints**: more nitrogen or more rain can never
  predict lower risk.
- **Search fast, verify slow.** The surrogate model scans 0–250 kg N/ha per zone for
  the smallest rate near peak yield with risk below 0.15; the full simulator then
  re-checks that rate and steps it down if the two disagree. Each zone also gets a
  plain-language advisory, such as splitting the dose around a forecast storm.
- **Calibration against real field trials.** A `calibrate` command fits
  quadratic-plateau yield curves to any nitrogen-rate trial CSV, such as the public
  PRNT dataset (49 US Midwest corn site-years), and retrains the model on the
  calibrated crop profile.
- **An advisor interface built for the field.** Draw or upload a field boundary on a
  satellite map, review color-coded zones, an NDVI layer, and the rate map, then
  export a zipped Shapefile with controller-safe attribute names, a point-grid CSV,
  and a JSON audit report recording every input, scene ID, and forecast.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Geospatial processing | Python 3.10+, GeoPandas, Rasterio, Shapely, PySTAC Client |
| Zone segmentation | Scikit-learn (StandardScaler, PCA, K-Means, silhouette scoring) |
| Process simulation | NumPy, SciPy (Monte Carlo water balance, curve fitting) |
| Nitrogen recommender | XGBoost (monotone-constrained dual-output regressor) |
| Advisor portal | Streamlit, Folium, streamlit-folium, Branca |
| Data sources | Sentinel-2 L2A (Earth Search), ISRIC SoilGrids 2.0, Open-Meteo |
| Storage | GeoTIFF, GeoJSON, Shapefile, Parquet (via PyArrow) with CSV fallback |

---

## Project Structure

```
.
├── spectral_engine.py    # Phase 1 — Sentinel-2 ingest, cloud masking, NDVI/NDWI/SAVI, synthetic generator
├── clustering.py         # Phase 2 — SoilGrids fusion, soil hydraulics, K-Means management zones
├── nitrogen_model.py     # Phase 3 — process simulator, XGBoost surrogate, N-rate optimizer
├── app.py                # Phase 4 — Streamlit / Folium advisor portal and exports
├── requirements.txt
├── out/                  # generated run outputs: indices, zones, prescriptions (not committed)
├── models/               # trained nitrogen models, one folder per crop (not committed)
└── cache/                # cached SoilGrids layers (not committed)
```

---

## Getting Started

### Prerequisites

- Python 3.10 or later
- No GPU required; everything runs on CPU
- Internet access only for **Live data** mode (the synthetic demo runs fully offline)

### Installation

```bash
git clone https://github.com/<your-username>/precision-nitrogen-platform.git
cd precision-nitrogen-platform

python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

Verify the install with each module's self-test:

```bash
python spectral_engine.py selftest
python clustering.py selftest
python nitrogen_model.py selftest
```

### Build the pipeline

The phases run **in order**; each writes artifacts the next one reads.

```bash
# Phase 1: generate a synthetic Sentinel-2 tile and compute spectral indices
python spectral_engine.py synthetic --out out/synthetic --stage vegetative --cloud 0.1

# Phase 2: segment the field into management zones (scored against synthetic ground truth)
python clustering.py synthetic --out out/zones_syn

# Phase 3: train the nitrogen model, then generate a prescription
python nitrogen_model.py train --crop corn --samples 40000 --out models/corn
python nitrogen_model.py recommend --model models/corn \
    --zones out/zones_syn/zone_profile.csv \
    --stage V8-V10 --prior-n 40 \
    --rain 0 3 32 14 0 --et0 5 5 3 3 5 \
    --out out/rx/rx.csv
```

To run on a real field instead, swap Phase 2 for live data and let Phase 3 pull the
forecast itself:

```bash
python clustering.py real --bbox -121.535 36.560 -121.515 36.575 \
    --start 2025-06-01 --end 2025-07-31 \
    --out out/field1 --cache cache/soilgrids

python nitrogen_model.py recommend --model models/corn \
    --zones out/field1/zone_profile.csv \
    --zones-geojson out/field1/zones.geojson \
    --stage V8-V10 --prior-n 40 --out out/field1/rx.csv
```

No environment variables are required. Earth Search, SoilGrids, and Open-Meteo are
public, unauthenticated endpoints. The app stores trained models in `models/` and
SoilGrids downloads in `cache/soilgrids/`, both relative to `app.py`.

---

## Usage

### Launch the advisor portal

```bash
streamlit run app.py
```

Opens at `http://localhost:8501`. The first run trains the nitrogen model if
`models/<crop>` doesn't exist yet; later runs load it from disk.

- **Sidebar** — choose synthetic or live data, crop and growth stage, nitrogen already
  applied, fertilizer product, live or what-if forecast, planned irrigation, and
  zoning settings
- **Boundary map** — draw a polygon or rectangle on satellite imagery, or upload a
  GeoJSON or zipped Shapefile
- **Results** — color-coded zone map with NDVI and rate layers, a prescription table,
  per-zone advisories, the forecast chart, zone soil profiles, and data provenance
- **Downloads** — zipped Shapefile (`N_KG_HA`, `N_LB_AC`, `PROD_LB_AC`, `RISK`),
  20 m point-grid CSV, zone-level CSV, and a JSON audit report

### Calibrate on real trial data

```bash
python nitrogen_model.py calibrate --trials prnt.csv \
    --site-col <SITE_YEAR_COL> --n-col <N_RATE_COL> --yield-col <YIELD_COL> \
    --yield-units kg_ha --out profiles/corn_prnt.json

python nitrogen_model.py train --crop corn \
    --calibrated-profile profiles/corn_prnt.json --out models/corn
```

### Model performance

All results below come from synthetic fields and simulation, not field trials.

| Zone segmentation (Phase 2, five synthetic fields) | Result |
|---|---|
| Sandy, high-leaching soil assigned to *High-Leaching Sand* | 90–97% |
| Loam assigned to a retention zone | 95–99% |
| Waterlogged swale assigned to *Saturated Lowland* | 27–100%, varies by field |
| Adjusted Rand index vs. ground-truth zones | 0.25–0.53 |

| Nitrogen surrogate (Phase 3, held-out simulations) | R² | MAE |
|---|---|---|
| Biomass gain head | 0.99 | ~120 kg DM/ha |
| Leaching-risk head | 0.97 | 0.03 |

| Prescription vs. flat rate (simulated 261 ha field, 4 zones) | Change |
|---|---|
| Nitrogen applied | 8% less |
| Nitrogen lost over the 5-day window | 14% less |
| Biomass gain | 0.5% lower |

When a large storm is forecast, the leaching-risk cap binds hard and the model cuts
each zone's rate well below its unconstrained optimum, recommending the remainder be
applied after the storm passes.

> ⚠️ **Advisory tool.** Rates should be verified against local agronomic guidance and
> in-season soil or tissue tests, within label and regulatory limits. The leaching
> equations are physically grounded but not yet field-calibrated.

---

## License

This project is licensed under the [MIT License](LICENSE) — see the `LICENSE` file
for details. (Swap in your preferred license before publishing if MIT isn't the
right fit for this repository.)
