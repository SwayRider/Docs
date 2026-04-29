# REQUIREMENT_005 — Highway & Street Names

## Overview

Add highway reference numbers and street names to the map with appropriate styling and progressive display based on zoom level.

## Context

- **Original Requirement**: 26_005
- **Components**: Data Pipeline, Map Styles
- **Priority**: Medium
- **Status**: Done

## Requirements

### Highway Names
- Show the highway letter-number combination on the map
  - In Europe highways have A123 numbering scheme, but some countries (like Belgium) also use E123 numbering scheme; we want to show both
  - A-number should be shown in a blue rectangle with white letters and a white border
  - E-number should be shown in a green rectangle with white letters and a white border
- The name boxes should be spread evenly across the map and if both A and E types are present alternate between them
- The boxes must be visible from Z7 onwards and should increase in number-displayed while zooming in
- Make sure they do not clutter the screen, but are plenty visible from Z11 onwards (they can be sparser above)
- Focus only on the highways in this requirement

### Street Names
- From Z15 onwards, show names for: Motorway, trunk, primary, secondary, tertiary (and their `_link` variants)
- From Z16 onwards, show names for: Unclassified, residential, living_street
- Font: Open Sans Regular
- Color: Black text with white halo/outline
- Orientation: Along the road line

## Acceptance Criteria

1. Highway A-numbers displayed in blue boxes
2. Highway E-numbers displayed in green boxes
3. Highway labels visible from Z7, increasing density with zoom
4. Street names visible from Z15 for major roads
5. Street names visible from Z16 for minor roads
6. Labels follow road orientation
7. Collision detection prevents label overlap

## Affected Files

### Data Pipeline
- `data-pipeline/build-tiles` — Road name extraction
- `data-pipeline/config/osmium-export-roads.json` — Verify `name` tag export

### Map Styles
- `assets/map/styles/light.json` — Highway and street label layers
- `assets/map/styles/dark.json` — Highway and street label layers
