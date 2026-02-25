#  Orbital EYE — Real-Time Satellite Tracker

> *Eye you can trust.*

A real-time, GPU-accelerated 2D satellite tracking visualizer built with Python and VisPy. Watch 11 satellites orbit Earth live on an interactive world map, with TLE data auto-refreshed from CelesTrak.

---

##  Features

- **Live orbit tracking** — Real-time lat/lon/altitude computed via SGP4 propagation
- **11 satellites preloaded** — ISS, Hubble, Tiangong, Starlink-1007, NOAA-20/21, GOES-18, Sentinel-2A, Terra, Aqua, Landsat-9
- **Orbital trail rendering** — Each satellite leaves a 120-point trail on the map
- **Auto TLE refresh** — Fetches fresh Two-Line Element sets from CelesTrak every 10 minutes; falls back to bundled TLEs offline
- **Side info panel** — Displays live position (lat/lon/alt), NORAD ID, orbital parameters, agency, mass, and more for the selected satellite
- **Interactive world map** — Custom land polygons, coastlines, equator line, and lat/lon grid rendered via VisPy visuals
- **Click to select** — Click any satellite dot to focus the info panel on it

---

##  Tracked Satellites

| Satellite     | NORAD  | Agency               | Orbit Type       | Altitude  |
|---------------|--------|----------------------|------------------|-----------|
| ISS           | 25544  | NASA/ESA/JAXA/Roscosmos/CSA | LEO     | ~408 km   |
| Hubble        | 20580  | NASA                 | LEO              | ~537 km   |
| NOAA-20       | 43013  | NOAA/NASA            | Polar (SSO)      | ~824 km   |
| Starlink-1007 | 44713  | SpaceX               | LEO              | ~550 km   |
| Tiangong      | 48274  | CNSA                 | LEO              | ~390 km   |
| Sentinel-2A   | 40697  | ESA                  | Polar (SSO)      | ~786 km   |
| GOES-18       | 51850  | NOAA/NASA            | GEO              | ~35,786 km|
| Terra         | 25994  | NASA                 | Polar (SSO)      | ~705 km   |
| Aqua          | 27424  | NASA                 | Polar (SSO)      | ~705 km   |
| Landsat-9     | 49260  | NASA/USGS            | Polar (SSO)      | ~705 km   |
| NOAA-21       | 54234  | NOAA/NASA            | Polar (SSO)      | ~824 km   |

---

##  Requirements

### Required
```bash
pip install vispy PyOpenGL PyOpenGL_accelerate pyqt5 numpy
```

### Recommended (for accurate SGP4 propagation)
```bash
pip install sgp4
```

### Optional (for high-quality land polygons)
```bash
pip install cartopy
```

### Optional (for live TLE fetching)
```bash
pip install requests
```

> Without `sgp4`, the tracker falls back to an approximate orbital model.  
> Without `requests`, bundled fallback TLEs are used.

---

##  Usage

```bash
python orbital_eye.py
```

The window opens at **1600×900** pixels. The map renders immediately using bundled TLEs. If `requests` is installed, live TLEs are fetched in the background from [CelesTrak](https://celestrak.org) and refreshed every 10 minutes.

### Controls

| Action               | Input              |
|----------------------|--------------------|
| Select satellite     | Left-click on dot  |
| Pan / zoom           | Mouse drag / scroll|

---

##  Project Structure

```
orbital_eye.py       # Single-file application — all logic included
```

| Component         | Description                                          |
|-------------------|------------------------------------------------------|
| `CATALOGUE`       | Satellite metadata (name, NORAD ID, color, specs)    |
| `FALLBACK_TLES`   | Bundled TLE strings for offline use                  |
| `fetch_live_tles` | Background thread — refreshes TLEs from CelesTrak    |
| `get_position`    | SGP4 propagation → lat / lon / altitude              |
| `LAND_POLYGONS`   | Hardcoded continent outlines for map rendering       |
| `SatTrack`        | Main VisPy app class — canvas, visuals, update loop  |

---

##  Visual Theme

The UI uses a dark terminal-green aesthetic:

| Element       | Color     |
|---------------|-----------|
| Ocean         | `#05111f` |
| Land          | `#1a3a28` |
| Land borders  | `#2ecc71` |
| Equator line  | `#1abc9c` |
| Grid lines    | `#0d2030` |
| UI text       | `#e0fff8` |

---

##  TLE Data Source

Live TLEs are fetched from:
```
https://celestrak.org/satcat/tle.php?CATNR=<NORAD_ID>
```
A status indicator in the panel shows whether data is `live` or `fallback`.

---
Visual Upgrades

<img width="1719" height="977" alt="Screenshot 2026-02-24 at 2 23 36 PM" src="https://github.com/user-attachments/assets/54f978ec-7b28-4dc2-a001-81230aed0e6f" />

---

##  Author

Built by **Gagandeep Singh Rathore**  
MCA (AI/ML) · Uttaranchal University
