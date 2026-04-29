# MILESTONE_001 — Foundation - Existing Requirements

## Overview

This milestone consolidates all existing requirements from the `@requirements` folder (26_001 through 26_009). These requirements cover foundational map features, service improvements, and pipeline refactoring needed before MVP development can proceed.

## Scope

- **Phase**: Pre-MVP
- **Priority**: High
- **Dependencies**: None (starting milestone)
- **Blocks**: MILESTONE_002, MILESTONE_003

## Requirements

| ID | Name | Components | Status |
|----|------|------------|--------|
| [REQUIREMENT_001](./MILESTONE_001/REQUIREMENT_001.md) | Served Map Styles | TilesService, TileViewer | In Progress |
| [REQUIREMENT_002](./MILESTONE_001/REQUIREMENT_002.md) | Add City Names | Data Pipeline, Map Styles | In Progress |
| [REQUIREMENT_003](./MILESTONE_001/REQUIREMENT_003.md) | Optimize Map Performance | Data Pipeline, Tiles | In Progress |
| [REQUIREMENT_004](./MILESTONE_001/REQUIREMENT_004.md) | Add Borders to L1/L2 | Data Pipeline, Map Styles | In Progress |
| [REQUIREMENT_005](./MILESTONE_001/REQUIREMENT_005.md) | Highway & Street Names | Data Pipeline, Map Styles | In Progress |
| [REQUIREMENT_006](./MILESTONE_001/REQUIREMENT_006.md) | Remove MinIO from Services | MailService, RegionService | In Progress |
| [REQUIREMENT_007](./MILESTONE_001/REQUIREMENT_007.md) | Mobile App Mapview & Search | Mobile App, TilesService | In Progress |
| [REQUIREMENT_008](./MILESTONE_001/REQUIREMENT_008.md) | Search Service | SearchService, Mobile App | In Progress |
| [REQUIREMENT_009](./MILESTONE_001/REQUIREMENT_009.md) | Pipeline Refactor | Data Pipeline | In Progress |

## Affected Components

### Backend Services
- **TilesService**: Map style serving (REQUIREMENT_001)
- **MailService**: MinIO removal (REQUIREMENT_006)
- **RegionService**: MinIO removal (REQUIREMENT_006)
- **SearchService**: New service for geocoding (REQUIREMENT_008)

### Data Pipeline
- **build-tiles**: Map optimization, city names, borders, highway/street names (REQUIREMENT_002, 003, 004, 005)
- **build-osm**: Pipeline refactor (REQUIREMENT_009)
- **build-border-data**: Pipeline refactor (REQUIREMENT_009)
- **build-valhalla-data**: Pipeline refactor (REQUIREMENT_009)
- **build-pelias-data**: Pipeline refactor (REQUIREMENT_009)

### Mobile App
- **Android**: Mapview integration, location search (REQUIREMENT_007, 008)

### Infrastructure
- **TileViewer**: Map style switching (REQUIREMENT_001)

## Dependencies Between Requirements

```
REQUIREMENT_001 (Map Styles) ──► REQUIREMENT_007 (Mobile Mapview)
         │
         ▼
REQUIREMENT_002 (City Names) ──┐
REQUIREMENT_003 (Map Optimize) ├─► REQUIREMENT_004 (Borders) ──► REQUIREMENT_005 (Highway/Street Names)
                               │
                               ▼
                        REQUIREMENT_009 (Pipeline Refactor)

REQUIREMENT_006 (MinIO Removal) ──► REQUIREMENT_008 (Search Service)
```

## Success Criteria

1. All map tiles render correctly with city names, borders, and road labels
2. Map rendering performance meets acceptable thresholds
3. TilesService serves map styles from filesystem
4. MailService and RegionService operate without MinIO dependency
5. SearchService provides centralized geocoding for mobile app
6. Data pipeline is modular with independent, runnable pipelines
7. Mobile app displays map with location search functionality

## Timeline Estimate

| Requirement | Estimated Effort | Notes |
|-------------|------------------|-------|
| REQUIREMENT_001 | 1-2 days | Style serving infrastructure |
| REQUIREMENT_002 | 2-3 days | Pipeline and style changes |
| REQUIREMENT_003 | 3-5 days | Analysis and optimization |
| REQUIREMENT_004 | 2-3 days | Border data extraction |
| REQUIREMENT_005 | 3-4 days | Highway and street labels |
| REQUIREMENT_006 | 2-3 days | Service refactoring |
| REQUIREMENT_007 | 3-5 days | Mobile app integration |
| REQUIREMENT_008 | 5-7 days | New service creation |
| REQUIREMENT_009 | 5-7 days | Pipeline restructuring |

**Total Estimated**: 26-39 days

## Open Questions

1. Are the existing analysis/improvement files in `@requirements/26_003/` complete?
2. What is the current status of each requirement (partially implemented, not started)?
3. Are there any blocking issues in the current implementations?
