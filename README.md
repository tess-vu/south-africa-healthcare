---
editor_options: 
  markdown: 
    wrap: sentence
---

# Geography of Healthcare Access in South Africa

## Potential Project Deliverables

**Client:** Distributed AI Research Institute (DAIR)

**Scope:** Piloting KwaZulu-Natal (KZN) & Gauteng provinces for future expansion to national scale.

## 1. Executive Summary

Scalable, dual-purpose geospatial system:

1.  **Public-Facing Dashboard:** Narrative-driven web map allowing the public to visualize healthcare inequities.

2.  **Research Backend:** Robust, verified geospatial database housed in Snowflake to support DAIR’s ongoing sociological and policy research.

## 2. Technical Architecture

**Frontend Visualization:** **Mapbox GL (?):** The client intends to scale from two provinces to the entire country.
Leaflet/OpenLayers will struggle with rendering thousands of pharmacy points and complex ward-level polygons simultaneously.
Mapbox GL’s vector tiling and WebGL rendering are helpful for this volume.

**Deployment:** Streamlit for rapid prototyping(?) or React for final production(?).

**Backend Data Engineering:** Snowflake

**Key Features:** Spatial Joins (`ST_WITHIN`) for point-in-polygon analysis, Python for ETL, Cortex (AI) for unstructured text parsing from PDF pharmacy lists.

## 3. Data Layers & Schema

The following datasets were noted in the touch-base meeting to be potentially engineered, normalized, and served to the dashboard.

### A. Core Demographics (Polygons & Rasters)

| Layer Name | Geometry | Source | Granularity | Fields | Notes |
|:-----------|:-----------|:-----------|:-----------|:-----------|:-----------|
| **Population (Coarse)** | Polygon | StatsSA Census 2022 | Province | `pop_count`, `pop_density`, `province_name` | Contextual basemap layer. |
| **Population (Fine)** | Polygon | StatsSA Census 2022 / SAL | Ward / Small Area | `pop_count`, `pop_density`, `ward_id`, `age_demographics` | Potential primary denominator for accessibility calculations. |
| **Population (Predicted)**\* | Raster/Grid | Engineered ML Model | \~100m Grid | `est_pop_count` | *Experimental Layer (See Section 6).* |
| **Legacy of Apartheid** | Polygon | DAIR Internal Data / Historical Maps | Neighborhood / Zone | `wealth_class` (Categorical) | **Classes:** 1. *Wealthy* (Suburb, Estate, Farm); 2. *Non-Wealthy* (Township, Informal Settlement); 3. *Non-Residential* (Industrial, Vacant) |

### B. Healthcare Supply (Points)

| Layer Name | Geometry | Source | Granularity | Fields | Notes |
|:-----------|:-----------|:-----------|:-----------|:-----------|:-----------|
| **Pharmacies** | Point | Engineered | Exact Lat/Lon | `pharmacy_id`, `name`, `type` (Private, Public, Informal), `is_registered` (Boolean), `nhi_funding_score` (0.0-1.0), `operating_hours` | *See Section 4 for NHI Score details.* |
| **Hospitals/Clinics** | Point | DAIR / Dept. of Health | Exact Lat/Lon | `facility_name`, `beds_count` (if any) | Necessary context. Public pharmacies are often inside clinics; Private are standalone. |

### C. Environmental & Mobility (Context)

| Layer Name | Geometry | Source | Granularity | Fields | Notes |
|:-----------|:-----------|:-----------|:-----------|:-----------|:-----------|
| **Greenery** | Polygon/Raster | DAIR / Sentinel-2 | Ward/Pixel | `ndvi_mean`, `green_view_index` | Proxy for neighborhood investment and mental health quality. Need to confirm granularity, metrics, and geometry type. |
| **Mobility Network** | MultiLine | OSM / GTFS | Road Segments | `road_type`, `is_taxi_route` (Boolean), `congestion_index` | This is assuming that these fields are as populated as western countries. |

## 4. NHI Index

*"Funding Likelihood" metric.*

The National Health Insurance (NHI) Act stipulates that funding is conditional on accreditation.
Rural and informal pharmacies often lack the infrastructure to qualify, creating a funding trap.

As an idea, it could be helpful to calculate a normalized `nhi_funding_score` (0.0 to 1.0) for each pharmacy point as this metric could allow the dashboard to toggle between "Physical Access" vs. "Financial Access":

-   **1.0 (High Likelihood):** Registered with SAPC + Located in Formal Commercial Zone (assuming South Africa has available zoning data) + Chain Affiliation + Brick and Mortar.
-   **0.5 (At Risk):** Registered with SAPC + Independent + Located in "Non-Wealthy" zone (potential infrastructure gaps).
-   **0.0 (Excluded):** Unregistered/Informal + Residential Zoning + No SAPC ID.

## 5. Routing & Accessibility Strategy

*Calculating "Distance".*

### OSMnx

Calculating network distance for millions of pairs using `osmnx` locally is computationally expensive and slow for an area as large as Gauteng/KZN.

**Proposed Approach:**

1.  **Pre-Calculation:** Do not calculate routes live in the browser.
    Pre-calculate "Service Areas" in the backend.

2.  **Engine:** Use Valhalla or OSRM (Open Source Routing Machine) running in a Docker container, or Snowflake's native geospatial functions for simple Euclidean buffers where road data is missing.

3.  **Modes of Transport:**

    *Pedestrian:* Frequent in Township/Informal areas.

    *Driving:* Relevant for Suburbs.

    *Minibus Taxi:* Minibus taxi routes are a popular transit mode.

    *Fallback:* If specific taxi routes are unavailable we could weight "Main Arterial Roads" higher in accessibility scoring.

## 6. Population Downscaling (Not in Original Scope)

*Refining Census data using ML that was briefly mentioned in touch-base meeting.*

**Problem:** Census Wards in rural areas or dense informal settlements can be spatially large or irregular, masking true distribution of people.
A pharmacy might be inside a ward, but far from where the people in that ward actually live.

Raesetje and Nyalleng mentioned ML.
We could train a Deep Learning model (like U-Net or a similar CNN architecture) to downscale coarse census counts into a high-resolution population density grid.

-   U-Net (Semantic Segmentation/Regression) is good at capturing context (urban texture) from imagery to predict pixel-level values.
-   **Inputs (Features):**
    -   **Satellite Imagery:** Sentinel-2 or SAR to detect structures.
    -   **Urban Proxies:** Building Footprints (Google/TUM), Impervious Surface layers, Road Density (OSM).
    -   Existing disaggregated Census Wards could help with verification.
-   **Output:** A predicted raster layer where pixel intensity = estimated population count.

## 7. Built Environment & Computer Vision (Not in Original Scope)

*Informal pharmacies.*

### A. Satellite Proxies

Since Census data in informal settlements is often outdated, we could use Building Footprints as a proxy for density, like from TUM's Global Building Atlas or Google Open Buildings.
So the metric would be `impervious_surface_ratio`.
This means High Imperviousness + Low Wealth Class = High Priority area for informal pharmacy scanning.

### B. Street View and Informal Pharmacies

-   **Concept:** Use Computer Vision to identify informal pharmacies via signage.
-   **Constraint:** Many informal pharmacies hide due to lack of permits or selling gray-market goods. They may operate inside general dealers or malls and may be without signage.
-   **Ethical Deliverable:** We could adjust coordinates or aggregate to a coarse hexagonal grid to show areas of informal access without pinpointing specific vulnerable vendors to law enforcement. Maybe actual coordinates would be in the backend admin access only, but that's still a security risk.
