# REQUIREMENT_002 — Add City Names

## Overview

Add city and town names to the map with progressive display based on zoom level, including visual indicators for city importance.

## Context

- **Original Requirement**: 26_002
- **Components**: Data Pipeline, Map Styles
- **Priority**: Medium
- **Status**: Done

## Requirements

- The city names should be added to the data pipeline
- The city names should appear progressively when zooming in, with larger cities shown earlier on
- The city names should draw on the map next to a square or dot indicating the center of the city/town
- The square/dot should represent the importance of the city/town and whether it's a capital or not
- The city names should start showing from zoom level 7 onwards (thus only in L1 and L2)

## Acceptance Criteria

1. City names appear on map at appropriate zoom levels
2. Larger cities/capitals appear at lower zoom levels (earlier)
3. Smaller towns appear at higher zoom levels (later)
4. Visual indicators (squares/dots) differentiate capitals from regular cities
5. Labels are readable and don't cause excessive clutter
6. City data is included in L1 and L2 MBTiles

## Affected Files

### Data Pipeline
- `data-pipeline/build-tiles` — City name extraction and tile generation
- `data-pipeline/config/` — Osmium export configuration

### Map Styles
- `assets/map/styles/light.json` — City label layers
- `assets/map/styles/dark.json` — City label layers

## Development Notes

- Use `assets/map/styles/dev-light.json` for development
- City importance can be derived from OSM `place` tag values (city, town, village)
- Capital status from OSM `capital` tag
