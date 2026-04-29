# SwayRider — Roadmap Index

This document provides an overview of all planned milestones for the SwayRider project, organized by development phase.

## Milestone Overview

| ID | Name | Phase | Status | Description |
|----|------|-------|--------|-------------|
| [MILESTONE_001](./MILESTONE_001.md) | Foundation - Existing Requirements | Pre-MVP | Done | Complete existing requirements (map styles, optimization, labels, search, pipeline) |
| [MILESTONE_002](./MILESTONE_002.md) | Infrastructure Cleanup | Pre-MVP | Done | Remove deprecated React web auth portal |
| [MILESTONE_003](./MILESTONE_003.md) | MVP Core Features | MVP | Planned | Route planning, turn-by-turn navigation, offline maps, POIs |
| [MILESTONE_004](./MILESTONE_004.md) | Auth Security Hardening | MVP (Pre-Public) | Planned | Critical and high-impact security improvements before public testers |
| [MILESTONE_005](./MILESTONE_005.md) | iOS/KMP Development | Post-MVP | Planned | iOS application using Kotlin Multiplatform shared logic |
| [MILESTONE_006](./MILESTONE_006.md) | Subscription & Payment | Post-MVP | Planned | Payment provider integration and subscription management |
| [MILESTONE_007](./MILESTONE_007.md) | Post-MVP Features | Post-MVP | Planned | Route sharing, group riding, community features |

## Phase Summary

### Phase 1: Pre-MVP (MILESTONE_001 - MILESTONE_002)
- Complete all existing requirements from the `@requirements` folder
- Remove deprecated web portal to clean up codebase
- Establish solid foundation for MVP development

### Phase 2: MVP (MILESTONE_003 - MILESTONE_004)
- Core routing and navigation features for Android
- Security hardening before opening to public testers
- Full Europe geographic coverage (already running on dev server)

### Phase 3: Post-MVP (MILESTONE_005 - MILESTONE_007)
- iOS application development (starts when Android is MVP ready)
- Subscription model and payment integration
- Community features (route sharing, group riding)

## Decision Points

| Decision | Status | Notes |
|----------|--------|-------|
| Payment Provider | TBD | To be selected during MILESTONE_006 |
| iOS Implementation Timeline | TBD | Starts after Android MVP (MILESTONE_003 complete) |
| Cloud Provider | TBD | MVP self-hosted; production evaluated at scale |
| Offline Map Download Strategy | TBD | Tile bundling vs on-demand download |
| POI Data Sources | TBD | OSM amenity tags vs third-party data |

## Key Dependencies

```
MILESTONE_001 ──► MILESTONE_002 ──► MILESTONE_003 ──► MILESTONE_004
                                        │
                                        ▼
                                    MILESTONE_005 ──► MILESTONE_006
                                        │
                                        ▼
                                    MILESTONE_007
```

- **MILESTONE_003** (MVP Core) blocks **MILESTONE_004** (Auth) and **MILESTONE_005** (iOS)
- **MILESTONE_005** (iOS) blocks **MILESTONE_006** (Subscription)
- **MILESTONE_007** (Post-MVP) can begin once MVP is complete

## Geographic Scope

| Phase | Coverage |
|-------|----------|
| MVP | Full Europe (currently running on dev server) |
| Post-MVP | Pan-European optimization and expansion |

## How to Use This Roadmap

1. Navigate to the specific milestone file for detailed requirements
2. Each milestone contains an INDEX.md with its requirements
3. Each requirement contains TASK_XX.md files for implementation details
4. Track progress by updating the Status column in this index
