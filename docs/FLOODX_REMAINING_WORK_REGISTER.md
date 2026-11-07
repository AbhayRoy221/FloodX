# FLOOD-X — REMAINING WORK REGISTER
## Prioritized Backlog of Incomplete, Simulated, Unverified, and Disconnected Components

- **Project:** FLOOD-X — AI-Powered Dynamic Flood-to-Action Decision Intelligence Platform
- **Target Basin:** Pune / Mula-Mutha River Basin, Maharashtra, India
- **Audit Date:** 6 November 2026
- **Auditor Role:** Lead Systems Architect & Scientific QA Reviewer
- **Priority Definitions:**
  - `P0` (Critical Blocker): Breaks startup, crashes runtime, causes silent test failure, or violates strict data integrity rules.
  - `P1` (High Priority): Disconnected operational workflow, missing core data layer, or unverified operational dependency.
  - `P2` (Medium Priority): Enhances fidelity from research baseline to operational deployment (e.g., native compilation, multi-basin ML).
  - `P3` (Low Priority): Documentation synchronization, minor refactoring, or maintenance cleanup.

---

## 1. Master Remaining Work Tasks

### TASK-ENV-01: Add Undocumented Geospatial Dependencies to `requirements.txt`
- **Phase / Category:** Phase 0 / Environment & Dependencies
- **Priority:** `P0`
- **Estimated Effort:** 1 hour (Estimate)
- **Why It Is Needed:** `backend/requirements.txt` currently omits `geopandas`, `shapely`, and `rasterio`. In a fresh environment, running `scripts/start-backend.ps1` or `pip install -r requirements.txt` succeeds silently, but importing `app.main` immediately fails with `ModuleNotFoundError: No module named 'geopandas'`.
- **Existing Implementation:** `requirements.txt` declares `fastapi`, `sqlalchemy`, `geoalchemy2`, `pyarrow`, `scikit-learn`, `xgboost`, `ortools`, etc.
- **Missing Implementation:** Explicit version pins for `geopandas>=0.14.0`, `shapely>=2.0.0`, `rasterio>=1.3.0`, and `pyogrio>=0.7.0` in `requirements.txt`.
- **Required Dataset or API:** None.
- **Dependencies:** None.
- **Files / Modules Requiring Changes:** `backend/requirements.txt`.
- **Acceptance Criteria:** `pip install -r requirements.txt` in a fresh Python 3.10 virtualenv installs all dependencies, and `python -c "from app.main import app"` succeeds without `ModuleNotFoundError`.
- **External Credentials Needed:** No.

---

### TASK-ENV-02: Eliminate Eager Top-Level Import in `osm_service.py`
- **Phase / Category:** Phase 1 / Code Architecture
- **Priority:** `P0`
- **Estimated Effort:** 2 hours (Estimate)
- **Why It Is Needed:** `backend/app/services/ingestion/osm_service.py` executes `import geopandas as gpd` at module top-level. Because `app/api/v1/router.py` eagerly imports all endpoint routers at startup, and `ingestion.py` imports `IngestionOrchestrator`, the entire FastAPI application crashes if `geopandas` is temporarily absent or slow to initialize.
- **Existing Implementation:** Top-level import in `backend/app/services/ingestion/osm_service.py`.
- **Missing Implementation:** Lazy-import `geopandas` inside `OSMIngestionService.ingest()` and `load_osm_geopackage()` methods.
- **Required Dataset or API:** None.
- **Dependencies:** None.
- **Files / Modules Requiring Changes:** `backend/app/services/ingestion/osm_service.py`.
- **Acceptance Criteria:** `app.main` can be imported cleanly even in environments where heavy spatial compilation packages are deferred.
- **External Credentials Needed:** No.

---

### TASK-NUGEN-01: Fix `TypeError` and Hardcoded Population Leak in `nugen_service.py`
- **Phase / Category:** Phase 4 / Phase 5 Integration
- **Priority:** `P0`
- **Estimated Effort:** 2 hours (Estimate)
- **Why It Is Needed:** In Phase 5.1, `estimated_exposed_population` was intentionally set to `None` due to the lack of census grid data. In `backend/app/services/resource_optimization/nugen_service.py` (line 206), the code executes `exposed_pop_str = f"{impact_res.population_exposure.estimated_exposed_population:,}"`. Because `None` cannot be formatted with `:,`, Python throws `TypeError: unsupported format string passed to NoneType.__format__`. This jumps to the `except Exception` block, which hardcodes `exposed_pop_str = "18,200"`, leaking an unsupported demographic number directly into the tactical advisory!
- **Existing Implementation:** Unhandled string formatting inside a broad `try/except` block with hardcoded fallback values.
- **Missing Implementation:** Check `if impact_res.population_exposure.estimated_exposed_population is not None` before formatting. If `None`, set `exposed_pop_str = "DATA INSUFFICIENT"`.
- **Required Dataset or API:** None.
- **Dependencies:** None.
- **Files / Modules Requiring Changes:** `backend/app/services/resource_optimization/nugen_service.py` (lines 205–213).
- **Acceptance Criteria:** Calling `POST /api/v1/nugen/advisory` executes without throwing exceptions and reports `DATA INSUFFICIENT` for population exposure instead of `18,200`.
- **External Credentials Needed:** No.

---

### TASK-NUGEN-02: Implement Live External Response Ingestion in `nugen_service.py`
- **Phase / Category:** Phase 4 / Cognitive Decision Support
- **Priority:** `P1`
- **Estimated Effort:** 1 day (Estimate)
- **Why It Is Needed:** Lines 260–277 of `nugen_service.py` make an HTTP POST to `${self._base_url}/advisory`. However, if the external service returns HTTP 200, the code only sets `provider_name = "Nugen Live Decision Intelligence API"` and completely **discards** `resp.json()`. The advisory text, observations, and action plan returned to the client are still 100% generated by the local python fallback template.
- **Existing Implementation:** HTTP client call with discarded response payload.
- **Missing Implementation:** Define external response schema, parse `resp.json()`, validate against Pydantic model, and map external executive summary, observations, and tactical actions into `DecisionAdvisoryResponse`. Add exponential backoff retry and 429 rate-limit handling.
- **Required Dataset or API:** Active Nugen API endpoint and valid API Key.
- **Dependencies:** Working external Nugen service.
- **Files / Modules Requiring Changes:** `backend/app/services/resource_optimization/nugen_service.py`, `backend/app/core/config.py`.
- **Acceptance Criteria:** When `NUGEN_API_KEY` is provided, the API payload from Nugen is parsed, validated, and returned in the response. If the call fails or times out, the NDMA SOP deterministic fallback is used with explicit status labeling.
- **External Credentials Needed:** **Yes** (Valid `NUGEN_API_KEY` and `NUGEN_API_BASE_URL`).

---

### TASK-MAP-01: Connect Overview GIS Map (`BasinLeafletMap.tsx`) to Live APIs
- **Phase / Category:** Phase 0 / Phase 1 Frontend UI
- **Priority:** `P1`
- **Estimated Effort:** 1.5 days (Estimate)
- **Why It Is Needed:** `frontend/src/components/map/BasinLeafletMap.tsx` currently displays static mock data: hardcoded array of 7 bridges, hardcoded array of 6 shelters with fabricated capacity numbers (e.g. 800, 500), and a static hardcoded polygon coordinate array. It never queries the backend API.
- **Existing Implementation:** Static Leaflet map with hardcoded in-file constants.
- **Missing Implementation:** Call `apiService.getGISFeatures()`, `apiService.getImpactLatest()`, and `apiService.getShelters()`. Dynamically render real PostGIS bridges, candidate shelters with `NULL` capacity badges, and dynamic flood footprints.
- **Required Dataset or API:** Backend endpoints `/api/v1/gis/features`, `/api/v1/impact/latest`, `/api/v1/evacuation/shelters`.
- **Dependencies:** Backend running with PostgreSQL.
- **Files / Modules Requiring Changes:** `frontend/src/components/map/BasinLeafletMap.tsx`.
- **Acceptance Criteria:** Overview map renders real geometries fetched from the backend API. Modifying a road closure or shelter in the database dynamically updates the overview map markers.
- **External Credentials Needed:** No.

---

### TASK-DATA-01: Ingest High-Resolution Gridded Demographic Population Raster
- **Phase / Category:** Phase 1 / Phase 5 GIS & Demographics
- **Priority:** `P1`
- **Estimated Effort:** 2 days (Estimate)
- **Why It Is Needed:** The repository currently has zero demographic rasters or census vectors. Population exposure numbers are suppressed as `DATA_INSUFFICIENT`. To provide actionable shelter planning and rescue dispatch, defensible population numbers are required.
- **Existing Implementation:** Schema returns `null` with `DATA_INSUFFICIENT_NO_INGESTED_CENSUS_GRID`.
- **Missing Implementation:** Ingest Pune district WorldPop 100m population raster (GeoTIFF) or Census of India 2011 Primary Census Abstract (PCA) ward shapefiles into PostGIS. Implement zonal statistics overlay in `impact_assessment_service.py` to calculate real pixel-sum exposed population.
- **Required Dataset or API:** WorldPop 100m Constrained Individual Countries (IND 2020) GeoTIFF or Pune Municipal Corporation Ward Boundary Shapefile.
- **Dependencies:** PostGIS raster extension or Python rasterstats.
- **Files / Modules Requiring Changes:** `backend/app/services/impact/impact_assessment_service.py`, `backend/app/schemas/impact.py`, `scripts/run_ingestion.py`.
- **Acceptance Criteria:** `POST /api/v1/impact/assess` returns physically calculated population counts with verifiable raster zonal pixel provenance.
- **External Credentials Needed:** No (WorldPop is open access).

---

### TASK-DATA-02: Acquire and Process Sentinel-1 Level-1 GRD SAR Rasters
- **Phase / Category:** Phase 1 / Phase 5 Remote Sensing
- **Priority:** `P2`
- **Estimated Effort:** 3 days (Estimate)
- **Why It Is Needed:** `data/raw/satellite/` does not exist. The current flood layer (`sentinel1_july2024_flood_extent.geojson`) is an illustrative polygon digitized from municipal flood reports. Raw SAR processing is needed for scientific reproducibility.
- **Existing Implementation:** Illustrative 3-polygon GeoJSON analog.
- **Missing Implementation:** Acquire Copernicus Sentinel-1 Level-1 GRD scenes (pre-event July 13, 2024, and post-event July 25, 2024). Build radiometric calibration, speckle filtering, and backscatter difference ($\Delta \sigma^0 \le -3.5\text{ dB}$) thresholding script.
- **Required Dataset or API:** ESA Copernicus Data Space Ecosystem (CDSE) Copernicus Sentinel-1 SAFE archives.
- **Dependencies:** GDAL / rasterio / snappy or Google Earth Engine.
- **Files / Modules Requiring Changes:** `scripts/acquire_sentinel1_sar.py`, `data/raw/satellite/`, `data/processed/satellite/`.
- **Acceptance Criteria:** Raw SAR TIFFs exist in `data/raw/satellite/`, and processed inundation masks are generated by an executable script with reproducible radiometric parameters.
- **External Credentials Needed:** **Yes** (Copernicus Data Space Ecosystem account).

---

### TASK-DATA-03: Establish Khadakwasla Dam Real-Time Telemetry Pipeline
- **Phase / Category:** Phase 1 / Phase 2 Hydrology
- **Priority:** `P1`
- **Estimated Effort:** 2 days (Estimate)
- **Why It Is Needed:** Maharashtra WRD does not stream live spillway discharge via an open unauthenticated REST API. Real-time riverine risk currently forces `DATA_INSUFFICIENT` unless an operator manually inputs a scenario discharge.
- **Existing Implementation:** `dam_telemetry_service.py` returns `DATA_INSUFFICIENT` in live mode and supports scenario sliders.
- **Missing Implementation:** Ingest official WRD departmental PDF bulletins via an automated scraper/OCR pipeline, or provide an authenticated administrative intake API (`POST /api/v1/telemetry/dam-discharge`) with auditable officer attribution.
- **Required Dataset or API:** Daily Khadakwasla Executive Engineer release bulletins.
- **Dependencies:** None.
- **Files / Modules Requiring Changes:** `backend/app/services/prospective/dam_telemetry_service.py`, `backend/app/api/v1/endpoints/prospective.py`.
- **Acceptance Criteria:** When a verified discharge value is ingested, `evaluate_riverine_risk()` transitions from `DATA_INSUFFICIENT` to active operational monitoring without requiring manual slider override.
- **External Credentials Needed:** No.

---

### TASK-SHELTER-01: Perform Municipal Structural Clearance on Candidate Shelters
- **Phase / Category:** Phase 3 / Evacuation Logistics
- **Priority:** `P1`
- **Estimated Effort:** Field Operational Task / 1 day data entry
- **Why It Is Needed:** All 64 shelters in the database have `total_capacity = NULL` and `verification_status = 'UNVERIFIED_FACILITY'`. The shelter recommendation engine returns `NO_VERIFIED_SHELTER_AVAILABLE`.
- **Existing Implementation:** Unverified candidate facility discovery and recommendation status handling.
- **Missing Implementation:** Ingest official municipal structural clearance certificates (`evidence_document`, `evidence_date`, `verified_by`) and certified bed capacities into the `shelters` table for at least 10 core municipal schools/halls.
- **Required Dataset or API:** Pune Municipal Corporation Disaster Management Cell gazetted shelter list.
- **Dependencies:** Database migration `59a8c1e2f3d4` (already applied).
- **Files / Modules Requiring Changes:** Database table `shelters` / `scripts/seed_shelters.py`.
- **Acceptance Criteria:** `POST /api/v1/evacuation/shelters/recommendations` returns status `SUCCESS` with certified shelters and non-zero available capacities.
- **External Credentials Needed:** No.

---

### TASK-RESOURCE-01: Ingest Verified Municipal Disaster Response Fleet
- **Phase / Category:** Phase 4 / Resource Optimization
- **Priority:** `P1`
- **Estimated Effort:** 1 day (Estimate)
- **Why It Is Needed:** 100% of the 23 records in `emergency_resources` are tagged `is_simulated = True`. OR-Tools optimization runs exclusively in simulation mode.
- **Existing Implementation:** Full CRUD and atomic allocation engine in `resource_inventory_service.py`.
- **Missing Implementation:** Ingest official PMC Fire Brigade, NDRF 5th Battalion (Sudumbare), and SDRF asset registers with verified depot locations, fleet capacities, and operational statuses (`is_simulated = False`).
- **Required Dataset or API:** PMC Fire Brigade / Disaster Cell fleet register.
- **Dependencies:** Database table `emergency_resources`.
- **Files / Modules Requiring Changes:** `backend/app/services/resource_optimization/resource_inventory_service.py`.
- **Acceptance Criteria:** `GET /api/v1/resources` reports non-zero verified assets (`is_simulated = False`), and OR-Tools can execute with `only_verified_resources = True`.
- **External Credentials Needed:** No.

---

### TASK-ML-01: Multi-Basin Expansion to Resolve Small Sample Size Limitation
- **Phase / Category:** Phase 2 / Machine Learning
- **Priority:** `P2`
- **Estimated Effort:** 4 days (Estimate)
- **Why It Is Needed:** The current ML model is trained on only 11 independent flood events across 5 years ($N_{\text{events}} = 11$, 39 positive days). With 5-fold cross-validation, each test fold evaluates only 2 to 3 events, producing high metric variance. It is strictly classified as `EXPERIMENTAL_EXPLORATORY_BASELINE`.
- **Existing Implementation:** Zero-leakage chronological and `GroupKFold` pipeline in `event_validation.py`.
- **Missing Implementation:** Expand training domain to include hydrologically similar adjacent Maharashtra river basins (Krishna at Sangli, Godavari at Nashik, Panchganga at Kolhapur) to scale sample size to $>50$ independent flood episodes.
- **Required Dataset or API:** IMD / CWC daily flood records and gridded reanalysis for adjacent Western Maharashtra basins.
- **Dependencies:** Feature engineering pipeline.
- **Files / Modules Requiring Changes:** `backend/app/services/ml/dataset_builder.py`, `backend/app/services/ml/event_validation.py`.
- **Acceptance Criteria:** Event cross-validation achieves stable confidence intervals across $\ge 50$ flood events with out-of-fold PR-AUC $>0.40$.
- **External Credentials Needed:** No.

---

### TASK-ML-02: Serialize ML Artifacts in Standard Cross-Platform Format
- **Phase / Category:** Phase 2 / Model Serialization
- **Priority:** `P2`
- **Estimated Effort:** 1 day (Estimate)
- **Why It Is Needed:** `floodx_xgboost_model.joblib` fails to unpickle when `xgboost` is not installed, and `floodx_rf_model.joblib` generates `InconsistentVersionWarning` (unpickled from scikit-learn 1.9.1 into 1.7.2). Pickling binary objects introduces fragile cross-version dependencies.
- **Existing Implementation:** `joblib.dump()` and `joblib.load()`.
- **Missing Implementation:** Save XGBoost in native JSON format (`model.save_model("model.json")`) and scikit-learn models using ONNX or pure array weight dictionaries.
- **Required Dataset or API:** Existing trained weights.
- **Dependencies:** `onnxruntime` or XGBoost JSON API.
- **Files / Modules Requiring Changes:** `backend/app/services/ml/model_trainer.py`, `backend/app/services/ml/inference.py`.
- **Acceptance Criteria:** Model weights can be loaded across different scikit-learn/Python minor versions without deserialization warnings or binary pickle crashes.
- **External Credentials Needed:** No.

---

### TASK-MOB-01: Integrate Interactive Map and Geolocation in Citizen Mobile Client
- **Phase / Category:** Phase 5 / Mobile Client
- **Priority:** `P2`
- **Estimated Effort:** 3 days (Estimate)
- **Why It Is Needed:** The mobile app (`mobile/`) currently lacks an interactive map (`react-native-maps` is absent) and GPS location tracking (`expo-location` is absent). Routes and shelters are presented only as text cards.
- **Existing Implementation:** Expo React Native UI with 7 screens and text-based route/shelter cards.
- **Missing Implementation:** Install `react-native-maps` and `expo-location`. Add an interactive vector map screen displaying the citizen's current GPS location, active river flood hazard buffers, closed road segments in red, and nearest safe shelters in green.
- **Required Dataset or API:** Mobile endpoints `/api/v1/evacuation/routes/plan` and `/api/v1/evacuation/shelters`.
- **Dependencies:** Expo SDK 51 compatibility.
- **Files / Modules Requiring Changes:** `mobile/package.json`, `mobile/src/screens/SafeRoutesScreen.tsx`, `mobile/src/screens/HomeScreen.tsx`.
- **Acceptance Criteria:** Citizen can view their real-time location on an interactive map, inspect turn-by-turn evacuation route geometries, and receive localized proximity warnings.
- **External Credentials Needed:** No.

---

### TASK-MOB-02: Configure Native Android Build Pipeline (EAS / Gradle)
- **Phase / Category:** Phase 5 / Mobile Deployment
- **Priority:** `P2`
- **Estimated Effort:** 2 days (Estimate)
- **Why It Is Needed:** The repository contains only a managed Expo workflow shell. `mobile/android/` does not exist, and no native APK or AAB binary has ever been compiled or tested on physical Android devices.
- **Existing Implementation:** Managed Expo TypeScript project (`app.json`, `package.json`).
- **Missing Implementation:** Run `npx expo prebuild` or configure Expo Application Services (EAS Build). Compile an installable release APK (`floodx-citizen-v0.1.0.apk`) and verify on an Android 13/14 physical test device.
- **Required Dataset or API:** Android SDK / Java 17.
- **Dependencies:** `TASK-MOB-01`.
- **Files / Modules Requiring Changes:** `mobile/app.json`, `mobile/eas.json`, `mobile/android/`.
- **Acceptance Criteria:** An installable `floodx-citizen.apk` binary is compiled and verified to install and communicate with the backend on a physical Android handset.
- **External Credentials Needed:** Expo / EAS account (optional, can build locally via Gradle).

---

### TASK-TEST-01: Implement Frontend & Mobile Automated Test Suites
- **Phase / Category:** Cross-Cutting / Quality Assurance
- **Priority:** `P2`
- **Estimated Effort:** 2 days (Estimate)
- **Why It Is Needed:** Frontend and mobile currently have zero unit, integration, or end-to-end automated tests. All quality verification has relied on TypeScript typechecking and production build exit codes.
- **Existing Implementation:** None.
- **Missing Implementation:** Install Vitest and React Testing Library in `frontend/`. Add test suites for API service handling, auth session token storage, CSRF token attachment, and UI component rendering.
- **Required Dataset or API:** Mock API responses.
- **Dependencies:** Node.js.
- **Files / Modules Requiring Changes:** `frontend/package.json`, `frontend/vite.config.ts`, `frontend/src/__tests__/`.
- **Acceptance Criteria:** `npm run test` executes in `frontend/` and validates core API integration, authentication, and error boundary handling.
- **External Credentials Needed:** No.

---

### TASK-REPLAN-01: Implement Event-Driven Background Re-Planning Engine
- **Phase / Category:** Phase 4 / Phase 5 Autonomous Operation
- **Priority:** `P2`
- **Estimated Effort:** 2.5 days (Estimate)
- **Why It Is Needed:** Re-planning currently requires manual user clicks. The final vision requires the platform to re-plan automatically when rainfall telemetry triggers a warning, river discharge changes, or a bridge is barricaded.
- **Existing Implementation:** On-demand REST APIs for scenario simulation and routing rebuilds.
- **Missing Implementation:** Implement a lightweight background worker (e.g. APScheduler or Celery) and WebSocket notification hub (`/ws/alerts`). When a new telemetry record or road closure is committed, the worker automatically re-runs prioritization, re-optimizes dispatch, and pushes updated advisories to connected authority dashboards and mobile clients.
- **Required Dataset or API:** Database change triggers or scheduled telemetry polling.
- **Dependencies:** FastAPI WebSockets.
- **Files / Modules Requiring Changes:** `backend/app/main.py`, `backend/app/services/evacuation/road_closure_service.py`, `frontend/src/services/api.ts`.
- **Acceptance Criteria:** Adding a road closure via API triggers an automatic recalculation event and broadcasts updated route guidance over WebSockets within 3 seconds.
- **External Credentials Needed:** No.

---

### TASK-DOC-01: Synchronize Phase 1 Data Integrity Manifest Checksums
- **Phase / Category:** Phase 1 / Governance & Integrity
- **Priority:** `P3`
- **Estimated Effort:** 1 hour (Estimate)
- **Why It Is Needed:** In `data/manifests/phase1_manifest.json`, the recorded SHA-256 hashes for `open_meteo_forecast_pune.json`, `open_meteo_historical_202407_pune.json`, and `chirps_v3_pune_daily_2020_2024_timeseries.csv` do not match current disk files because these files were updated during Phase 2.
- **Existing Implementation:** Outdated manifest generated on 2026-11-04.
- **Missing Implementation:** Re-run `scripts/generate_manifest.py` (read-only verification first) to document updated file checksums and explain the exact revision history in `docs/DATA_QUALITY_RULES.md`.
- **Required Dataset or API:** Current repository files.
- **Dependencies:** None.
- **Files / Modules Requiring Changes:** `data/manifests/phase1_manifest.json`.
- **Acceptance Criteria:** All files listed in `phase1_manifest.json` match disk SHA-256 hashes exactly.
- **External Credentials Needed:** No.

