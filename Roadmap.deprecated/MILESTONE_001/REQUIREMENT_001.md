# REQUIREMENT_001 — Served Map Styles

## Overview

The TilesService should serve map styles from files in a configured directory, allowing easy updates, additions, and removal of styles without code changes.

## Context

- **Original Requirement**: 26_001
- **Components**: TilesService
- **Priority**: High
- **Status**: Done

## Requirements

### TilesService

- The tilesservice serves map styles from files in a configured directory
- There are 2 default styles, which are always available (light and dark)
- Extra styles always come in 2 variants (light and dark); they are named: `<name>-light` and `<name>-dark`
- We have an endpoint to get all available styles
- We create an extra folder in the root of the repository: `assets/map/styles` where we put the source of the map styles
- For debugging we point the tilesservice to the `assets/map/styles` folder. In production we will deploy the styles explicitly

## Acceptance Criteria

1. TilesService reads styles from configured directory at startup
2. `/v1/tiles/styles` endpoint returns list of available styles
3. `/v1/tiles/styles/{name}` endpoint returns style JSON
4. Default styles `light` and `dark` are always available
5. Custom styles follow `<name>-light` and `<name>-dark` naming convention

## Affected Files

### Backend
- `backend/services/tilesservice/` — Style serving logic

### Assets
- `assets/map/styles/` — Style JSON files
