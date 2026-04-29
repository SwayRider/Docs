# REQUIREMENT_009 — Pipeline Refactor

## Overview

Refactor the monolithic data pipeline into smaller, independent pipelines focused on specific tasks with shared configuration and resources.

## Context

- **Original Requirement**: 26_009
- **Components**: Data Pipeline
- **Priority**: High
- **Status**: Done

## Background

The current `build` pipeline is monolithic, performing multiple steps. We want to split it into smaller, focused pipelines that can run independently or together.

## Requirements

### Pipeline Structure

1. **build-osm**: Prepares OSM data
   - Create overlap extracts
   - Create region OSM files

2. **build-border-data**: Prepares border data
   - Extract region borders
   - Create region border areas
   - Detect border crossings
   - Depends on build-osm

3. **build-valhalla-data**: Prepares Valhalla data
   - Create Valhalla routing data
   - Depends on build-osm

4. **build-pelias-data**: Prepares Pelias data
   - Create Pelias geocoding data
   - Export from Elasticsearch for cluster migration
   - Depends on build-osm

5. **build-tiles**: Prepares MBTiles
   - Already in place, should not be touched

### Configuration
- Unified config file format across all pipelines
- Shared tools, downloads, and temporary directories
- Each pipeline prepares its own prerequisites

### Packaging
- Each pipeline finishes with packaging output into tar file:
  - build-osm: `osm.tar.bz2`
  - build-border-data: `border.tar.bz2`
  - build-valhalla-data: `valhalla.tar.bz2`
  - build-pelias-data: `pelias.tar.bz2`
  - build-tiles: `tiles.tar` (keep as is)

## Acceptance Criteria

1. Each pipeline runs independently (with prerequisites checked)
2. Unified configuration format across pipelines
3. Shared resource directories work correctly
4. Each pipeline produces expected tar output
5. Dependency checking prevents running without prerequisites
6. All pipelines runnable from MacBook

## Affected Files

### Data Pipeline
- `data-pipeline/build-osm` — New or refactored
- `data-pipeline/build-border-data` — New or refactored
- `data-pipeline/build-valhalla-data` — New or refactored
- `data-pipeline/build-pelias-data` — New or refactored
- `data-pipeline/build-tiles` — Unchanged
- `data-pipeline/config/` — Unified configuration
