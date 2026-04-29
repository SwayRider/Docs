# REQUIREMENT_007 — Mobile App Mapview & Search

## Overview

Add MapLibre mapview to the mobile app home screen and implement location search functionality.

## Context

- **Original Requirement**: 26_007
- **Components**: Mobile App, TilesService
- **Priority**: High
- **Status**: Done

## Requirements

### Mapview
- Add a MapLibre map to the home screen
- Add build config fields to set the server URL
  - For debug: `https://tiles.hevanto-it.com`
  - Reference the TileViewer for service connection
- Fetch tiles and the default light style from the service
- Center on the current location of the user at Z12

### Location Search
- Search for locations using Pelias services
- Resolve search regions using RegionService
- Display dropdown with results when multiple matches found
- No autocomplete at this moment
- Search box at top of screen over the map with search icon
- Clicking search icon or pressing Enter starts the search
- Progress spinner while searching

### Search Flow
1. Resolve search regions using RegionService `search-box` endpoint
2. Query Pelias for each region returned
3. Merge and rank results (core-region priority, then confidence)
4. Display up to 5 results
5. No-results fallback: retry with all remaining Pelias servers
6. Error handling: show snackbar on network failure

## Acceptance Criteria

1. Map displays on home screen with MapLibre
2. Tiles load from configured server URL
3. Map centers on user location at Z12
4. Search box visible at top of map
5. Search queries RegionService and Pelias
6. Results display in dropdown
7. Selecting result places marker and pans map
8. Zoom level determined by Pelias layer type

## Affected Files

### Mobile App
- `mobile/android/app/` — Mapview and search implementation
- `mobile/android/app/build.gradle.kts` — Build config
