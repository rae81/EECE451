# Network Cell Analyze

Android client + Flask backend + LightGBM dead-zone prediction service for Lebanese cellular coverage (Alfa, Touch — 2G/3G/4G/5G).

The Android app collects live cell measurements (RSRP, SNR, operator, network type, neighbor cells, GPS) and streams them to the Flask server. The server stores them in SQLite, fuses them with OpenCellID towers, Ookla speed-test tiles, OpenStreetMap features and SRTM elevation, and serves a dual-head LightGBM model that scores any `(latitude, longitude, operator, network_type)` point in Lebanon for dead-zone risk.

---

## Repository layout

```
android-app/NetworkCellAnalyzer/   Android Studio project (Java)
server/                            Flask + ML backend
├── app.py                         REST routes, ingest, dashboard, SocketIO
├── models.py                      SQLAlchemy schema
├── config.py                      Environment-driven config
├── deadzone_model.py              Dual-head model class + serving
├── deadzone_features.py           43-feature schema (38 numeric, 5 categorical)
├── deadzone_data.py               Dataset fusion (towers + Ookla + OSM + DEM + app)
├── deadzone_physics.py            COST-231 Hata, FSPL, propagation helpers
├── deadzone_propagation.py        ITU-R P.1812 / Bullington diffraction
├── deadzone_weak_supervision.py   Snorkel-style soft-label generator
├── deadzone_training.py           Spatial-CV training pipeline
├── deadzone_explain.py            SHAP TreeExplainer reasoning
├── deadzone_transfer.py           Berlin V2X cross-city evaluation
├── lebanon_geometry.py            Land-polygon mask for prediction grids
├── device_identity.py             MAC-OUI vendor resolver
├── data/                          OpenCellID, Ookla, OSM, DEM, coastline
├── instance/ml/                   Trained model bundle + reports
├── templates/, static/            Dashboard HTML/CSS
└── tests/                         Pytest suite
figures/                           Architecture, model and UI screenshots
```

---

## Architecture

### Android client
- Java, `minSdk 24`, `compileSdk 34`
- `CellMonitorService` — foreground service polling `TelephonyManager.getAllCellInfo()` plus fused-location every 10 s
- Room + WorkManager — local store and 15-minute offline-sync flush
- Retrofit 2 + OkHttp 3 — REST; Socket.IO client for live dashboard
- Google Maps SDK + Leaflet WebView — heatmap modes (Signal · Towers · Operator · Reliability · Dead-zone prediction)
- AndroidX Biometric — optional fingerprint unlock

### Flask backend
- Flask 3.x · Flask-SocketIO (threading) · Flask-CORS · Flask-SQLAlchemy
- SQLite (default) — switch by setting `DATABASE_URL`
- `itsdangerous` signed bearer tokens (24 h access / 30 d refresh)
- ReportLab PDF export, streaming CSV export
- Gunicorn (`gunicorn.conf.py`) for production; `python app.py` for dev

### Dead-zone model
- Dual LightGBM heads sharing 43 features
  - **regressor** — Huber loss → predicted RSRP (dBm)
  - **classifier** — binary log-loss → P(dead zone)
- 38 numeric features: location, tower topology, COST-231 / FSPL / P.1812 propagation physics, SRTM-derived terrain, Ookla performance, OSM urban context, and app-collected H3-res9 aggregates
- 5 categorical features: operator, network type, H3-res7, H3-res9, urban class
- 5 weak-supervision labeling functions combined by accuracy-weighted log-odds majority voting; tiered hard thresholds for direct labels
- Spatial GroupKFold by H3-res5 hex (~253 km²) with a 2 km buffer between train and test points
- Optuna hyperparameter search (PR-AUC objective on spatial CV)
- SHAP `TreeExplainer` produces top-3 human-readable reasons per prediction

Trained metrics (Lebanon hold-out): ROC-AUC 0.998, recall 0.996, F1 0.889, RSRP RMSE 11.5 dB. Zero-shot evaluation on the Fraunhofer HHI BerlinV2X dataset: RSRP RMSE ≈ 10 dB.

---

## Server setup

```bash
cd server
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python3 app.py        # http://localhost:5000
```

Tests:

```bash
cd server
source .venv/bin/activate
pip install -r requirements-dev.txt
pytest
```

Retrain the model bundle:

```bash
cd server
flask --app app retrain-model
```

The serialized model lives at `server/instance/ml/deadzone_model.joblib` and is loaded once at Flask startup. Six of the 43 features (`app_*`) are recomputed per request from recent SQLite rows, so predictions reflect live uploads even between retrains.

---

## Android setup

1. Open `android-app/NetworkCellAnalyzer` in Android Studio.
2. Set the server base URL in **Settings** inside the app (defaults to `http://192.168.0.139:5000/`, configurable at runtime).
3. Run on a physical device — the cellular APIs (`TelephonyManager`, `READ_PHONE_STATE`, foreground location) are not meaningful in an emulator.
4. Grant the requested runtime permissions: `READ_PHONE_STATE`, `ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`, `POST_NOTIFICATIONS`.

---

## Selected REST endpoints

See `server/API_CONTRACT.md` for the full contract.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/auth/login`, `/auth/register`, `/auth/refresh` | Bearer-token auth |
| `POST` | `/api/cell/ingest`, `/api/cell/ingest/batch` | Measurement ingest |
| `GET`  | `/predict?latitude&longitude&operator&network_type` | Single dead-zone score |
| `POST` | `/predict/batch` | Up to 200 grid points (Lebanon-masked) |
| `GET`  | `/api/stats/device`, `/api/stats/fleet` | Aggregates |
| `GET`  | `/api/heatmap-data` | Map buckets |
| `GET`  | `/api/tower-clusters[/detail]` | Observed-cell DBSCAN clusters |
| `GET`  | `/api/export.csv`, `/api/report.pdf` | Exports |

---

## Database schema

`users` · `cell_data` · `neighbor_cell_data` · `device_log` · `speed_test_results` · `alert_rules`. SQLAlchemy models in `server/models.py`; `db.create_all()` runs at boot.

---

## Deployment

- `Procfile` and `render.yaml` for Render
- `gunicorn.conf.py` for worker / thread sizing
- `instance/` is gitignored except for the shipped model bundle and training reports — keep your local SQLite out of the repo
