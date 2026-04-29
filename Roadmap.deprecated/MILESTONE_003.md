# MILESTONE_003 — MVP Core Features

## Overview

Implement the core features required for the SwayRider MVP, including route planning, turn-by-turn navigation, offline maps, and points of interest for the Android mobile application.

## Scope

- **Phase**: MVP
- **Priority**: Critical
- **Dependencies**: MILESTONE_001, MILESTONE_002
- **Blocks**: MILESTONE_004, MILESTONE_005

## Requirements

| ID | Name | Components | Status |
|----|------|------------|--------|
| [REQUIREMENT_001](./MILESTONE_003/REQUIREMENT_001.md) | Route Planning | RouterService, Mobile App | Planned |
| [REQUIREMENT_002](./MILESTONE_003/REQUIREMENT_002.md) | Turn-by-Turn Navigation | Mobile App | Planned |
| [REQUIREMENT_003](./MILESTONE_003/REQUIREMENT_003.md) | Offline Maps | Mobile App, TilesService | Planned |
| [REQUIREMENT_004](./MILESTONE_003/REQUIREMENT_004.md) | Points of Interest | Mobile App, Data Pipeline | Planned |

## Affected Components

### Backend Services
- **RouterService**: Multi-region routing with Valhalla integration
- **TilesService**: Offline map download support

### Mobile App
- **Route Planning UI**: Origin/destination selection, route preferences
- **Navigation UI**: Turn-by-turn instructions, voice guidance
- **Map Download**: Offline tile management
- **POI Layer**: Points of interest display and interaction

### Data Pipeline
- **POI Data**: Extraction of amenities from OSM

## Success Criteria

1. Users can plan routes with customizable preferences (highway vs scenic, toll avoidance)
2. Turn-by-turn navigation works with voice guidance
3. Maps can be downloaded for offline use
4. Points of interest are visible and searchable
5. All features work across full Europe coverage
6. Performance meets acceptable thresholds on target devices

## Timeline Estimate

| Requirement | Estimated Effort |
|-------------|------------------|
| REQUIREMENT_001 | 3-4 weeks |
| REQUIREMENT_002 | 4-5 weeks |
| REQUIREMENT_003 | 3-4 weeks |
| REQUIREMENT_004 | 2-3 weeks |
| **Total** | **12-16 weeks** |

## Key Decisions Needed

1. Route preference UI design
2. Voice guidance implementation (TTS vs pre-recorded)
3. Offline map download strategy (tile bundling vs on-demand)
4. POI data sources (OSM vs third-party)
5. Navigation offline capability scope
