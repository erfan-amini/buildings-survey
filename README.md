# NSI Field Survey Tool

A browser-based geospatial survey application for collecting and managing building inventory data across multiple study areas. The tool integrates field verification workflows with automated building attribute estimation, backed by Google Sheets for collaborative data management.

---

## Overview

The NSI Field Survey Tool supports hazard vulnerability assessments by enabling field teams to verify, correct, and supplement building-level data from the National Structure Inventory (NSI). It provides a map interface for navigating to structures, recording survey observations, and synchronizing results to a shared backend in real time.

The tool currently supports four study areas: **Shinnecock**, **Mastic Beach**, **Pamunkey**, and **West Point**.

---

## Survey Workflow

Each building in the inventory exists in one of three visual states:

| Marker Color | State | Meaning |
|:---:|---|---|
| 🔵 Blue | Verify | Pre-populated NSI record requiring field confirmation |
| 🔴 Red | New Survey | Point added during fieldwork, not in original NSI |
| 🟢 Green | Surveyed | Completed survey with a recorded `savedAt` timestamp |

A surveyor selects a point on the map, reviews or enters the building attributes, and saves. Each save writes immediately to the backend. The **Undo Save** action resets a point to its pre-survey state without deleting the underlying row.

---

## Automated Attribute Estimation

### Building Footprint Area

Footprint area can be auto-detected using the **Microsoft Building Footprints** dataset, queried via the ArcGIS REST API. When the surveyor clicks **🏠 Auto** next to the Footprint field:

1. A spatial intersection query is sent to the Microsoft Building Footprints Feature Service using the point's WGS84 coordinates.
2. If a building polygon is found at that location, its area is computed using the **Shoelace formula** with geodesic correction (degree-to-meter projection at the local latitude).
3. The result, in square feet, is populated into the Footprint field.

The footprint area represents the ground-level building outline. Gross floor area is derived internally as `footprint × number_of_stories` wherever needed (e.g., for cost estimation).

### Ground Elevation

Ground surface elevation (ft, NAVD88) can be auto-fetched from the **USGS 3D Elevation Program (3DEP)** via the Elevation Point Query Service (EPQS). When the surveyor clicks **📐 Auto** next to the Ground Elevation field:

1. A point query is sent to the USGS EPQS v1 API with the building's WGS84 coordinates.
2. The service returns the interpolated bare-earth elevation from the 3DEP dynamic elevation layer, which uses **1-meter resolution lidar-based DEMs** where available, falling back to 1/3 arc-second (~10m) seamless DEMs elsewhere.
3. The result, in feet relative to NAVD88, is populated into the Ground Elevation field.

The overall accuracy of the USGS elevation service has an RMSE of approximately 0.53 meters, though this varies by location and source data resolution.

### Structure and Content Value Estimation

Structure replacement cost is estimated using **Ordinary Least Squares (OLS) linear regression** trained on the local building stock. When the surveyor clicks **💰 Auto** next to the value fields:

**Step 1 — Reference Data Collection.** All buildings in the current study area that have both a known gross floor area and a known structure value are collected. Gross area is computed as `footprint_area × stories` for each building.

**Step 2 — Model Fitting.** If at least 5 reference buildings are available, an OLS linear regression is fit:

```
V̂_structure = β₁ × A_gross + β₀
```

where `A_gross` is gross floor area (sqft), `V̂_structure` is the predicted structure replacement value (USD), `β₁` (slope) represents the marginal cost per additional square foot, and `β₀` (intercept) captures size-independent baseline costs. The coefficients are computed via standard OLS normal equations. A sanity check rejects implausibly low predictions and falls back to the median method.

**Step 3 — Prediction.** The model is applied to the target building's gross area to produce the structure value estimate.

**Step 4 — Content Value.** Content replacement value is set to half the structure value:

```
V_content = V̂_structure / 2
```

**Fallback behavior:**
- **2–4 reference buildings:** The median cost per gross square foot across the reference set is used instead of regression.
- **Fewer than 2 reference buildings:** No estimate is produced; the surveyor must enter values manually.

This approach ensures that cost estimates reflect **local construction costs and property characteristics** rather than relying on national look-up tables. The model is recalculated each time the button is pressed, so it incorporates any data collected during the current session.

---

## Data Architecture

### Frontend

The application is a single-page web app built with React, Leaflet (mapping), and Babel (in-browser JSX transpilation). It runs entirely in the browser with no build step. All building data is pulled from the Google Sheets backend on load — no building records are embedded in the frontend code.

### Backend

Google Apps Script serves as a lightweight REST API. Each study area is stored in a dedicated pair of Google Sheets tabs:

| Tab | Purpose |
|---|---|
| `Surveys_<location>` | One row per building with all survey attributes |
| `DevEdits_<location>` | JSON blob storing developer-mode geometry edits |

Supported API operations: **Load** (GET all rows for a location), **Save** (upsert by UID), **Delete** (remove by UID), **Bulk Save** (full rewrite), **Get/Save Dev Edits** (geometry edit state).

### Data Schema

Each building record contains 20 fields:

| Field | Description |
|---|---|
| `uid` | Unique identifier (e.g., `nsi-46007` or `new-10051`) |
| `survey_type` | `verify` (NSI record) or `survey` (new point) |
| `ID` | Numeric building ID |
| `occupancy_type` | FEMA/Hazus occupancy class (e.g., `RES1-2SNB`, `COM4`) |
| `building_type` | Construction: W (Wood), M (Masonry), C (Concrete), S (Steel), H (Manufactured) |
| `number_of_stories` | Story count |
| `area` | Building footprint area (sqft) |
| `foundation_type` | S (Slab), C (Crawlspace), B (Basement), P (Pier/Pile), W (Solid Wall) |
| `foundation_height` | First floor height above grade (ft) |
| `year_built` | Construction year |
| `ground_elevation` | Ground surface elevation (ft, NAVD88) |
| `address` | Street address |
| `longitude` | WGS84 longitude |
| `latitude` | WGS84 latitude |
| `structure_value` | Structure replacement cost (USD) |
| `content_value` | Content replacement cost (USD) |
| `basement` | Yes / No |
| `notes` | Free-text field observations |
| `surveyor` | Surveyor name |
| `savedAt` | ISO 8601 timestamp (empty = not yet surveyed) |

### Building ID Convention

NSI buildings retain their original numeric IDs. New buildings added through the tool receive sequential IDs starting at **10001**, auto-incremented from the highest existing new-point ID.

---

## Developer Mode

An optional developer mode provides geometry editing capabilities:

| Action | Description |
|---|---|
| **Add Point** | Click the map to place a new survey point with an auto-generated ID |
| **Move Point** | Relocate a selected point; updated coordinates sync to the sheet |
| **Remove Point** | Delete a point from the map and backend |

Developer edits are stored separately and can be cleared via the **Clear Edits** button, which resets both local and server-side edit state. Note that synced deletions (rows already removed from the sheet) cannot be restored through this action — use **Pull Sheet** to reload from the backend.

---

## Synchronization

| Direction | Button | Behavior |
|---|---|---|
| Sheet → App | **⬇ Pull Sheet** | Overwrites local state from the Google Sheet. Clears all local dev edits. Use when starting a session or after manual sheet edits. |
| App → Sheet | **☁️ Sync Sheet** | Pushes all local data to the sheet, overwriting its contents for the current location. |

Background polling refreshes survey data and geometry edits every 30 seconds to reflect changes made by other users.

---

## External Data Sources

| Source | Usage | Access |
|---|---|---|
| [Microsoft Building Footprints](https://github.com/Microsoft/USBuildingFootprints) | Automated footprint area detection | ArcGIS REST API (public, no auth) |
| [USGS 3DEP Elevation](https://www.usgs.gov/3d-elevation-program) | Ground elevation (ft, NAVD88) from 1m lidar DEMs | EPQS REST API (public, no auth) |
| [National Structure Inventory](https://www.hec.usace.army.mil/confluence/nsidocs) | Baseline building attributes | Pre-loaded into Google Sheets |
| [OpenStreetMap](https://www.openstreetmap.org) | Base map tiles | Public tile server |

---

## Citation

If using this tool or its methodology in academic work, please cite:

> Amini, E. (2025). *NSI Field Survey Tool: A web-based building inventory verification and attribute estimation platform.* GitHub repository.

---

## License

This project is provided for research and educational use. Microsoft Building Footprints data is licensed under the [Open Data Commons Open Database License (ODbL)](https://opendatacommons.org/licenses/odbl/). OpenStreetMap tiles are © OpenStreetMap contributors.
