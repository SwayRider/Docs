# REQUIREMENT_004 — Add Borders to L1 and L2

## Overview

Add country border rendering to L1 and L2 zoom levels. Currently borders are only rendered on L0 (world level).

## Context

- **Original Requirement**: 26_004
- **Components**: Data Pipeline, Map Styles
- **Priority**: Medium
- **Status**: Done

## Requirements

- Detailed OSM borders on L1 and L2
- L1 borders simplified, L2 not
- Viewer should render the borders

## Acceptance Criteria

1. Country borders visible on L1 tiles (simplified)
2. Country borders visible on L2 tiles (detailed)
3. Border rendering does not significantly impact tile size
4. Borders are correctly aligned across tile boundaries
5. Style differentiates borders from other features

## Affected Files

### Data Pipeline
- `data-pipeline/build-tiles` — Border data extraction
- `data-pipeline/pipeline/tiles.py` — Border layer generation

### Map Styles
- `assets/map/styles/light.json` — Border layer styling
- `assets/map/styles/dark.json` — Border layer styling

## Development Notes

- Use `assets/map/styles/light.json` for development
- Pipeline code in `data-pipeline/` folder
