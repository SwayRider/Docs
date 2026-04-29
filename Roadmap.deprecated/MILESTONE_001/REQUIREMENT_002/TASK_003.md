# TASK_003 — Zoom Level Configuration

**Status**: Done

## Overview

Configure progressive display of place labels based on city importance, with larger cities and capitals appearing at lower zoom levels and smaller settlements at higher zoom levels.

## Repository

- **Repo**: swayrider
- **Subfolder**: `data-pipeline/pipeline/`, `assets/map/styles/`
- **Tech**: Python (ogr2ogr SQL), MapLibre style JSON

## Scope: Existing

- `data-pipeline/pipeline/tiles.py` — `add_places_minzoom()` function
- `assets/map/styles/light.json` — minzoom on each layer
- `assets/map/styles/dark.json` — minzoom on each layer

## Technical Specification

### Data Pipeline Minzoom (`tiles.py`)

`add_places_minzoom()` injects minzoom via ogr2ogr SQL:

| Place Type | L1 Minzoom | L2 Minzoom |
|------------|------------|------------|
| city | 7 | 11 |
| town | 8 | 11 |
| village | 10 | 11 |
| hamlet | 10 | 12 |
| suburb | — | 13 |
| neighbourhood | — | 14 |

### Style Layer Minzoom

Each style layer has a `minzoom` property matching the data minzoom:

| Layer | Minzoom |
|-------|---------|
| places-capital-square | 7 |
| places-dot-city-major | 8 |
| places-dot-city-minor | 9 |
| places-labels-city-major | 8 |
| places-labels-city-minor | 9 |
| places-dot-town / places-labels-town | 10 |
| places-dot-village / places-labels-village | 12 |
| places-dot-hamlet / places-labels-hamlet | 14 |

### Population-Based Filtering

- City major: no population data OR population ≥ 25000
- City minor: population < 25000
- Capitals: always shown from Z7 regardless of population

## Dependencies

- TASK_001 (minzoom data must be in tiles)
- TASK_002 (style layers must exist)

## Acceptance Criteria

- [x] Capitals visible from Z7
- [x] Major cities visible from Z8
- [x] Minor cities visible from Z9
- [x] Towns visible from Z10
- [x] Villages visible from Z12
- [x] Hamlets visible from Z14
- [x] Progressive density increase while zooming

## Testing Notes

- Verify minzoom values in generated GeoJSON with `ogrinfo`
- Check style rendering at each zoom breakpoint
- Confirm no label clutter at Z7–Z10
