# REQUIREMENT_004 — Points of Interest

## Overview

Display and enable searching for points of interest relevant to motorcycle riders, including restaurants, fuel stations, scenic viewpoints, rest areas, and repair shops.

## Context

- **Components**: Mobile App, Data Pipeline
- **Priority**: Medium
- **Status**: Planned

## Requirements

### POI Categories
| Category | OSM Tags | Display Icon |
|----------|----------|--------------|
| Fuel Stations | amenity=fuel | Gas pump |
| Restaurants | amenity=restaurant | Fork/knife |
| Rest Areas | highway=rest_area | Bench |
| Scenic Viewpoints | tourism=viewpoint | Camera |
| Repair Shops | shop=motorcycle | Wrench |
| Parking | amenity=parking | P |

### Map Display
- POI icons visible on map at appropriate zoom levels
- Category filtering (show/hide categories)
- POI detail popup on tap
- Search nearby functionality

### Search
- Search by name or category
- Filter by distance from route
- Filter by open hours
- Sort by distance or rating (if available)

### Data Source
- OSM amenity tags as primary source
- Consider third-party data for enrichment (future)

## Acceptance Criteria

1. POI categories display on map
2. Category filtering works
3. POI details shown on tap
4. Search finds POIs by name
5. Nearby search filters by distance
6. POI data included in offline maps
7. Icons scale appropriately with zoom

## Affected Files

### Data Pipeline
- `data-pipeline/` — POI data extraction from OSM

### Mobile App
- `mobile/android/app/src/main/java/.../ui/poi/` — POI display and search
- `mobile/android/app/src/main/java/.../data/poi/` — POI data layer

### Map Styles
- POI icon layers in map styles
