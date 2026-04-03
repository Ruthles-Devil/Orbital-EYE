#  Orbital EYE — Real-Time Satellite Orbital Tracking System

> **A Technical Research Paper**
> Graduate Level — Aerospace & Software Engineering
>
> **Author:** Gagandeep Singh Rathore
> **Year:** 2025

---

## Abstract

This paper presents **Orbital EYE**, a real-time satellite orbital tracking and visualization system developed as a personal engineering project. The system integrates live Two-Line Element (TLE) data retrieval from Celestrak, SGP4 orbital propagation, and GPU-accelerated 2D rendering using the VisPy framework to display the real-time positions, ground tracks, and telemetry of eleven active satellites spanning Low Earth Orbit (LEO), Sun-Synchronous Orbit (SSO), and Geostationary Orbit (GEO).

The architecture employs a multi-threaded design to decouple live data acquisition from rendering, achieving smooth real-time updates at a 2-second refresh interval. Interactive telemetry panels expose spacecraft metadata, live geodetic coordinates, altitude, and mission context. The system demonstrates how modern Python tooling can bridge orbital mechanics and high-performance visualization in a lightweight, dependency-minimal desktop application.

**Key contributions include:**
- A graceful fallback propagation model for SGP4-unavailable environments
- Polygon-based continental rendering with optional high-resolution Cartopy coastlines
- A pixel-space HUD overlay architecture decoupled from the geographic scene graph

**Keywords:** `satellite tracking` `SGP4` `TLE` `VisPy` `real-time visualization` `orbital mechanics` `LEO` `GEO` `ground track` `Python`

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Background and Related Work](#2-background-and-related-work)
3. [System Architecture](#3-system-architecture)
4. [Satellite Catalogue](#4-satellite-catalogue)
5. [Implementation Details](#5-implementation-details)
6. [Results and Performance](#6-results-and-performance)
7. [Discussion](#7-discussion)
8. [Conclusion](#8-conclusion)
9. [References](#9-references)

---

## 1. Introduction

The proliferation of artificial satellites over the past six decades has made near-Earth orbital space one of the most consequential and congested operational environments in human history. As of 2025, more than 8,000 active satellites orbit Earth, performing functions ranging from weather observation and broadband communications to Earth imaging and scientific research. Understanding the real-time disposition of these assets requires accessible, accurate, and interactive tools that can bridge the gap between raw orbital data and meaningful human comprehension.

**Orbital EYE** was developed as a personal graduate-level engineering project with the goal of creating a self-contained, cross-platform desktop application capable of fetching live orbital element sets, propagating satellite positions in real time, and rendering those positions on an interactive world map with contextual telemetry overlays. The project synthesizes multiple disciplines: astrodynamics, software architecture, real-time rendering, and human-computer interaction.

### 1.1 Motivation

Existing satellite tracking solutions either require web connectivity with server-side computation (e.g., Heavens-Above, Celestrak web interface), or are complex commercial packages with steep learning curves. Orbital EYE occupies a middle ground: a lightweight Python application that performs all computation locally, operates gracefully in offline or degraded-network scenarios via fallback TLE caching, and presents results through an intuitive, visually polished interface.

### 1.2 Scope and Contributions

This paper makes the following technical contributions:

- A **multi-threaded architecture** separating TLE acquisition, orbital propagation, and GPU rendering into independent execution contexts
- A **dual-mode propagation system** supporting both rigorous SGP4 and a simplified analytical fallback for dependency-constrained environments
- A **pixel-space HUD overlay system** built on VisPy's scene graph, enabling resolution-adaptive telemetry panels without distorting the geographic coordinate system
- A **satellite catalogue** of eleven scientifically and operationally significant spacecraft spanning three orbit regimes (LEO, SSO, GEO)
- **Graceful degradation** and fallback strategies at every layer of the system stack

---

## 2. Background and Related Work

### 2.1 Two-Line Element Sets (TLE)

A Two-Line Element set is a standardized data format encoding the orbital parameters of an Earth-orbiting object at a specific epoch. Developed by NORAD and maintained by Space-Track.org and Celestrak, TLEs encode mean orbital elements including inclination, right ascension of the ascending node (RAAN), eccentricity, argument of perigee, mean anomaly, and mean motion, among other quantities.

Orbital EYE retrieves TLEs directly from Celestrak's per-satellite API endpoint, refreshing every **600 seconds**. When live retrieval fails, the system falls back to a bundled set of recent TLEs embedded in the application source.

```
https://celestrak.org/satcat/tle.php?CATNR=<NORAD_ID>
```

### 2.2 SGP4 Orbital Propagation

The **Simplified General Perturbations 4 (SGP4)** model is the standard algorithm for propagating NORAD TLE-based orbital elements forward in time. SGP4 accounts for:

- Atmospheric drag
- Solar radiation pressure (via B* drag term)
- Earth's oblateness (J2 through J4 zonal harmonics)

It produces Earth-Centered Inertial (ECI) Cartesian coordinates that are subsequently transformed to geodetic latitude and longitude by applying the Greenwich Sidereal Time (GST) rotation.

The Python `sgp4` library (Brandon Rhodes, 2012–present) implements the Vallado C++ reference propagator with full double-precision arithmetic.

### 2.3 VisPy Rendering Framework

VisPy is a Python library providing high-performance interactive 2D/3D visualizations through direct access to OpenGL via PyOpenGL. It offers a scene graph abstraction (`vispy.scene`) that supports multiple camera types, hierarchical visual objects, and event-driven interaction.

Orbital EYE uses VisPy's `PanZoomCamera` for the geographic view and constructs a secondary pixel-space camera for the HUD overlay, both sharing the same `SceneCanvas`.

### 2.4 Related Systems

| System | Type | Propagation | Offline? | Open Source |
|---|---|---|---|---|
| Heavens-Above | Web | Server-side SGP4 | No | No |
| Celestrak Viewer | Web | Server-side SGP4 | No | No |
| Gpredict | Desktop (C) | SGP4 | Yes | Yes |
| Stellarium + plugin | Desktop (C++) | SGP4 | Partial | Yes |
| **Orbital EYE** | **Desktop (Python)** | **SGP4 + fallback** | **Yes** | **Personal** |

*Table 1. Comparison of Orbital EYE against related satellite tracking systems.*

---

## 3. System Architecture

Orbital EYE is structured around three loosely coupled subsystems: the **Data Acquisition Layer**, the **Propagation Engine**, and the **Visualization & Interaction Layer**. These subsystems communicate through shared in-memory data structures protected by threading primitives.

### 3.1 High-Level Architecture

| Layer | Component | Technology | Thread |
|---|---|---|---|
| Data Acquisition | TLE Fetcher | `requests` + Celestrak API | Background daemon |
| Data Acquisition | TLE Store | `dict` + `threading.Lock` | Shared (RW-locked) |
| Propagation | SGP4 Engine | `sgp4.api` (Satrec) | Main / timer callback |
| Propagation | Fallback Model | Analytical sinusoidal approx. | Main / timer callback |
| Visualization | Map Renderer | VisPy `scene.visuals` | Main (OpenGL) |
| Visualization | HUD Overlay | VisPy pixel-space camera | Main (OpenGL) |
| Visualization | Trail History | In-memory deque (120 pts) | Main |

*Table 2. Architectural layers, components, and threading model.*

### 3.2 Data Acquisition Layer

A background daemon thread continuously refreshes TLE data for all catalogued satellites at **600-second intervals**. Each satellite is fetched individually via NORAD catalogue number to ensure per-asset freshness. Fetched TLEs are validated by checking line prefix characters (`'1'` and `'2'`) and stored in a thread-safe dictionary (`TLE_STORE`) protected by a `threading.Lock`.

On fetch failure, the in-memory store retains the most recently valid TLE, and the application displays a `fallback` status indicator.

### 3.3 Propagation Engine

Position computation is invoked every **two seconds** via VisPy's application timer. For each catalogued satellite, the `get_position()` function:

1. Acquires a read lock on `TLE_STORE`
2. Parses the current TLE pair using `Satrec.twoline2rv()`
3. Computes the Julian Date (JD + fractional day) for the current UTC instant
4. Calls `s.sgp4(jd, fr)` to obtain ECI position vector **r = [x, y, z]** in kilometres

The **ECI-to-geodetic conversion** applies the Greenwich Sidereal Time (GST) formula to rotate the inertial x-y plane to the Earth-fixed frame, yielding geodetic longitude. Latitude is derived from the elevation angle of the position vector above the equatorial plane. Altitude above the WGS-84 mean sphere (R = 6371 km) is computed as the vector magnitude minus Earth's radius.

### 3.4 Fallback Propagator

When the `sgp4` library is unavailable, Orbital EYE employs an analytical approximation:

```
T = 86400 / mean_motion          # orbital period in seconds
ν = 2π × t / T                   # mean anomaly at current epoch
lat = arcsin(sin(i) × sin(ν))    # geodetic latitude
lon = f(ν, t)                    # longitude (linear advance)
```

This model provides plausible ground-track shapes but does not account for precession or perturbations, and is presented as a degraded-mode fallback only.

### 3.5 Visualization Layer

The VisPy `SceneCanvas` hosts two independent view objects:

- **Geographic View (`vmap`):** Uses `PanZoomCamera` with fixed aspect ratio, locked to the geographic coordinate system. Hosts the world map, coastlines, grid, equator, satellite trail lines, marker sprites, and name labels.
- **HUD View (`vhud`):** Uses `PanZoomCamera` in pixel-space with y-axis flipped (screen origin at top-left). Hosts all telemetry panel widgets. Panel geometry is rebuilt on every window resize event.

---

## 4. Satellite Catalogue

Orbital EYE tracks eleven satellites selected to represent a diversity of orbit regimes, operator agencies, mission types, and orbital characteristics.

| Satellite | NORAD ID | Agency | Altitude | Inclination | Mission Type |
|---|---|---|---|---|---|
| ISS | 25544 | Multi-agency | ~408 km | 51.6° | Crewed Station |
| Hubble (HST) | 20580 | NASA | ~537 km | 28.5° | Space Telescope |
| NOAA-20 | 43013 | NOAA/NASA | ~824 km | 98.7° | Weather/Env. |
| Starlink-1007 | 44713 | SpaceX | ~550 km | 53.0° | Broadband Comms |
| Tiangong CSS | 48274 | CNSA | ~390 km | 41.5° | Crewed Station |
| Sentinel-2A | 40697 | ESA | ~786 km | 98.6° | Earth Obs. |
| GOES-18 | 51850 | NOAA/NASA | ~35,786 km | 0.0° | Geostationary Wx |
| Terra | 25994 | NASA | ~705 km | 98.2° | Climate/EO |
| Aqua | 27424 | NASA | ~705 km | 98.2° | Water Cycle |
| Landsat-9 | 49260 | NASA/USGS | ~705 km | 98.2° | Land Imaging |
| NOAA-21 | 54234 | NOAA/NASA | ~824 km | 98.7° | Weather/Env. |

*Table 3. Orbital EYE satellite catalogue with key orbital and mission parameters.*

The catalogue spans three distinct orbit regimes:
- **LEO** (300–2000 km): ISS, Tiangong, Starlink, Hubble, Sentinel, Terra, Aqua, Landsat, NOAA
- **SSO** (retrograde near-polar LEO): NOAA-20/21, Sentinel-2A, Terra, Aqua, Landsat-9
- **GEO** (~35,786 km): GOES-18 — appears stationary over the Americas on the map display

---

## 5. Implementation Details

### 5.1 Map Rendering

The world map is constructed programmatically from two sources:

1. **Fallback polygons** — Hand-crafted vertex lists covering eleven regions: North America, South America, Europe, Africa, Asia, Japan/Korea, Australia, New Zealand, Greenland, Iceland, and Antarctica. Each polygon is rendered as a closed line strip.
2. **Cartopy (optional)** — When available, upgrades to Natural Earth 50m-resolution coastline shapefiles for significantly improved geographic accuracy.

Grid lines are drawn at **30-degree intervals** in both axes. The equator and prime meridian receive distinct styling to provide spatial reference.

### 5.2 Trail Rendering

Each satellite maintains a **ring-buffer trail history** of up to 120 position samples (~4 minutes at 2-second intervals). The trail uses linearly interpolated alpha values from **6% (oldest)** to **80% (newest)**, creating a fading-tail effect.

Anti-meridian discontinuities (crossing ±180° longitude) are handled by inserting `NaN` sentinel values that break the line strip, preventing spurious horizontal lines across the map.

### 5.3 HUD Telemetry Panel

The interactive telemetry panel is divided into three sections:

| Section | Content | Update Rate |
|---|---|---|
| Spacecraft | Agency, launch date, mass, altitude, inclination, period, speed | Static (on selection) |
| Live Position | Geodetic lat/lon, altitude, NORAD ID | Every 2 seconds |
| Mission Notes | Descriptive text, mission-specific data | Static (on selection) |

A per-satellite **color accent stripe** and **header dot** are updated on selection to provide visual identity continuity between the map marker and the telemetry panel.

### 5.4 Interaction Model

| Input | Action |
|---|---|
| Left click on satellite | Select — opens telemetry panel |
| Left click on empty area | Deselect |
| Escape key | Deselect / close panel |
| Scroll | Zoom in/out |
| Drag | Pan the map |
| Window resize | Rebuilds HUD panel geometry |

Click events compute the map-space coordinate via VisPy's inverse transform, then perform a **nearest-neighbor search** within a 5-degree tolerance radius.

---

## 6. Results and Performance

### 6.1 Orbital Accuracy

Using live TLEs and the full double-precision SGP4 propagator, Orbital EYE achieves positional accuracy consistent with published SGP4 performance:

- **Typical error:** 1–2 km RMS for LEO satellites at short propagation intervals
- **TLE refresh:** Every 600 seconds, ensuring element set age remains well within SGP4's accuracy range
- **GOES-18 calibration:** Its near-stationary behavior on the map serves as a visual coordinate system sanity check

### 6.2 Rendering Performance

On a mid-range development workstation, Orbital EYE maintains smooth interactive frame rates. Each 2-second tick performs:

- 11 SGP4 propagations (Julian Date computation + `sgp4()` call each)
- Trail buffer updates for all active satellites
- Marker position and color updates
- Conditional telemetry panel refresh

The most computationally intensive operation is the **initial map construction**, which occurs once at startup.

### 6.3 Fallback Mode Behavior

In SGP4-unavailable environments, the fallback propagator:

- ✅ Correctly preserves orbital inclination as peak latitude envelope
- ✅ Preserves approximate orbital period
- ❌ Accumulates systematic longitude errors over time
- ❌ Does not model J2 precession or atmospheric drag

The terminal status line clearly indicates propagator mode (`SGP4` or `simplified`) so operators are aware of accuracy level.

---

## 7. Discussion

### 7.1 Design Decisions

**Why VisPy over Matplotlib or PyGame?**
VisPy's OpenGL backend handles hundreds of visual objects (coastline segments, trail histories, markers, labels) without frame rate degradation. Matplotlib's rendering pipeline introduces unacceptable latency for real-time updates; PyGame lacks a scene graph abstraction suitable for geographic coordinate systems.

**Why dual-camera architecture?**
Placing fixed-size UI elements in geographic coordinates would cause them to scale and translate unpredictably during zoom/pan operations. Separating the coordinate systems allows each camera to optimize for its own domain — geographic fidelity for the map, pixel-perfect placement for the HUD.

### 7.2 Limitations

- **Projection:** Simple equirectangular (plate carrée) projection introduces area and shape distortions at high latitudes
- **Fallback accuracy:** The simplified propagator does not account for J2 nodal precession or atmospheric drag
- **Static catalogue:** Hardcoded satellite list; no dynamic constellation management
- **No prediction:** Pass times, AOS/LOS for ground stations not yet implemented

### 7.3 Future Work

- 🌍 **3D globe rendering** mode using VisPy's 3D scene graph
- 📡 **Ground station markers** with real-time azimuth/elevation computation
- ⏰ **Satellite pass prediction** and event alerting (AOS, LOS, maximum elevation)
- 🛰️ **Dynamic constellation expansion** supporting full Starlink, OneWeb, or Iridium catalogues
- 📁 **Export functionality** for KML/GeoJSON ground track overlays
- 🔐 **Space-Track.org integration** for classified NORAD catalogue access

---

## 8. Conclusion

This paper presented **Orbital EYE**, a real-time satellite tracking and visualization system developed as a personal graduate engineering project. The system successfully integrates live TLE acquisition, SGP4 orbital propagation, and GPU-accelerated geographic rendering into a self-contained Python desktop application with interactive telemetry display for eleven active spacecraft.

The architecture demonstrates several principled engineering decisions:
- Separation of data acquisition from rendering via threading
- Graceful degradation through a fallback propagation model
- Adaptive HUD rendering through a decoupled pixel-space coordinate system
- Anti-meridian-aware trail rendering

Orbital EYE illustrates the accessibility of modern astrodynamics tooling: with the `sgp4` library, VisPy, and a handful of supporting dependencies, a single developer can construct a tracking system of meaningful accuracy and visual quality. The project serves as both a practical engineering demonstration and a personal exploration of the intersection between orbital mechanics and interactive software systems.

---

## 9. References

1. F. R. Hoots and R. L. Roehrich, "Models for Propagation of NORAD Element Sets," *Spacetrack Report No. 3*, Aerospace Defense Command, 1980.
2. D. A. Vallado, P. Crawford, R. Hujsak, and T. S. Kelso, "Revisiting Spacetrack Report #3: Rev 1," *AIAA 2006-6753*, 2006.
3. T. S. Kelso, *Celestrak Satellite Catalog and TLE Archive*. [Online]. Available: https://celestrak.org. Accessed: 2025.
4. B. Rhodes, *sgp4 Python Library*. [Online]. Available: https://pypi.org/project/sgp4/. Accessed: 2025.
5. VisPy Development Team, "VisPy: Interactive Scientific Visualization in Python." [Online]. Available: https://vispy.org. Accessed: 2025.
6. Natural Earth Data, "Natural Earth 1:50m Physical Vectors." [Online]. Available: https://www.naturalearthdata.com. Accessed: 2025.
7. NASA, "International Space Station Overview." *NASA Technical Reports Server*, 2024.
8. NOAA/NASA, "Joint Polar Satellite System (JPSS) Mission Overview." *NOAA Technical Report*, 2022.
9. ESA, "Sentinel-2 User Handbook." *ESA Standard Document*, Issue 1, Rev. 2, 2015.
10. D. A. Vallado, *Fundamentals of Astrodynamics and Applications*, 4th ed. Microcosm Press / Springer, 2013.

---

<div align="center">

*Orbital EYE — Graduate Technical Report · Gagandeep Singh Rathore · 2025*

</div>
