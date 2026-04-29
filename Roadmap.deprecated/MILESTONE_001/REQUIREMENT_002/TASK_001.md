# TASK_001 — City Data Extraction

**Status**: Done

## Overview

Extract city, town, and village data from OpenStreetMap for inclusion in L1 and L2 MBTiles, with importance tags (place, population, capital).

## Repository

- **Repo**: swayrider
- **Subfolder**: `data-pipeline/`
- **Tech**: Python, Osmium, GDAL/ogr2ogr

## Scope: Existing

- `data-pipeline/pipeline/osm_funcs.py` — Osmium filters for place nodes/relations
- `data-pipeline/pipeline/tiles.py` — places.pbf → places.geojson conversion + minzoom injection
- `data-pipeline/config/osmium-export-places.json` — Tag export config

## Technical Specification

### OSM Extraction (`osm_funcs.py`)

Both L1 and L2 extracts include:
```
n/place=city n/place=town n/place=village n/place=hamlet n/place=suburb n/place=neighbourhood
r/place=city r/place=town r/place=village r/place=hamlet r/place=suburb r/place=neighbourhood
```

### Export Config (`osmium-export-places.json`)

```json
{
  "include_tags": ["place", "name", "population", "capital", "admin_level"]
}
```

### GeoJSON Conversion (`tiles.py`)

- `places.pbf` → `places.geojson` with `--geometry-types=point`
- `add_places_minzoom()` adds minzoom property via ogr2ogr SQL:
  - L1: city=7, town=8, village=10, hamlet=10
  - L2: city=11, town=11, village=11, hamlet=12, suburb=13, neighbourhood=14

## Dependencies

- None (first task in chain)

## Acceptance Criteria

- [x] Place nodes extracted from OSM for L1 and L2
- [x] Tags exported: place, name, population, capital, admin_level
- [x] GeoJSON contains point geometries only
- [x] minzoom property added based on place type

## Testing Notes

- Verify extraction with `ogrinfo` on places.geojson
- Check minzoom values match expected zoom levels
