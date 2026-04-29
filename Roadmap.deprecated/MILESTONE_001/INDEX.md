# MILESTONE_001 — Requirements Index

**Status**: Done

This milestone contains 9 requirements covering foundational map features, service improvements, and pipeline refactoring.

## Requirements

| ID | Name | Components | Original Requirement | Status |
|----|------|------------|----------------------|--------|
| [REQUIREMENT_001](./REQUIREMENT_001.md) | Served Map Styles | TilesService | 26_001 | Done |
| [REQUIREMENT_002](./REQUIREMENT_002.md) | Add City Names | Data Pipeline, Map Styles | 26_002 | Done |
| [REQUIREMENT_003](./REQUIREMENT_003.md) | Optimize Map Performance | Data Pipeline, Tiles | 26_003 | Done |
| [REQUIREMENT_004](./REQUIREMENT_004.md) | Add Borders to L1/L2 | Data Pipeline, Map Styles | 26_004 | Done |
| [REQUIREMENT_005](./REQUIREMENT_005.md) | Highway & Street Names | Data Pipeline, Map Styles | 26_005 | Done |
| [REQUIREMENT_006](./REQUIREMENT_006.md) | Remove MinIO from Services | MailService, RegionService | 26_006 | Done |
| [REQUIREMENT_007](./REQUIREMENT_007.md) | Mobile App Mapview & Search | Mobile App, TilesService | 26_007 | Done |
| [REQUIREMENT_008](./REQUIREMENT_008.md) | Search Service | SearchService, Mobile App | 26_008 | Done |
| [REQUIREMENT_009](./REQUIREMENT_009.md) | Pipeline Refactor | Data Pipeline | 26_009 | Done |

## Requirement Groups

### Map Rendering & Styles
- REQUIREMENT_001: Map style serving infrastructure
- REQUIREMENT_002: City name labels on map
- REQUIREMENT_003: Map rendering performance optimization
- REQUIREMENT_004: Country border rendering
- REQUIREMENT_005: Highway and street name labels

### Service Infrastructure
- REQUIREMENT_006: Remove MinIO dependency from MailService and RegionService
- REQUIREMENT_008: Create new SearchService for centralized geocoding

### Mobile Application
- REQUIREMENT_007: Integrate MapLibre mapview and location search

### Data Pipeline
- REQUIREMENT_009: Refactor monolithic pipeline into modular pipelines

## Progress Tracking

To track progress on individual requirements, refer to their respective INDEX.md files:
- [REQUIREMENT_001/INDEX.md](./REQUIREMENT_001/INDEX.md)
- [REQUIREMENT_002/INDEX.md](./REQUIREMENT_002/INDEX.md)
- [REQUIREMENT_003/INDEX.md](./REQUIREMENT_003/INDEX.md)
- [REQUIREMENT_004/INDEX.md](./REQUIREMENT_004/INDEX.md)
- [REQUIREMENT_005/INDEX.md](./REQUIREMENT_005/INDEX.md)
- [REQUIREMENT_006/INDEX.md](./REQUIREMENT_006/INDEX.md)
- [REQUIREMENT_007/INDEX.md](./REQUIREMENT_007/INDEX.md)
- [REQUIREMENT_008/INDEX.md](./REQUIREMENT_008/INDEX.md)
- [REQUIREMENT_009/INDEX.md](./REQUIREMENT_009/INDEX.md)
