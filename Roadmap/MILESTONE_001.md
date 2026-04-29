# Milestone 001 - Foundation Complete

**Status**: Done

## Goals

Establish the full technical foundation of the SwayRider platform. This includes all map rendering infrastructure, backend services, geocoding, the mobile app mapview with search, and data pipeline refactoring. All work in this milestone is complete.

## Key Deliverables

### Map Rendering Infrastructure

TilesService serves vector map styles from filesystem. Map tiles render with city names, administrative borders at L1/L2 level, highway names, and street-level labels. Rendering performance is optimized across all zoom levels.

### Backend Services

All six Go backend microservices are operational: AuthService (JWT auth, user management), MailService (SMTP, no MinIO dependency), RegionService (spatial queries, border crossings, no MinIO dependency), RouterService (multi-region Valhalla routing), SearchService (Pelias-backed geocoding with multi-region fan-out), TilesService (vector tile and style serving).

### Search Service

SearchService implements a 4-phase Pelias geocoding flow with address collapsing, ranking, and result labeling. Configured via `PELIAS_REGIONS=region=url,...`. Replaces direct Pelias calls from the mobile app.

### Mobile App Foundation

Android app (Kotlin, Jetpack Compose, Clean Architecture) displays vector map tiles via MapLibre. Location search is integrated against SearchService with JWT authentication. Dark theme, remember-me, and forgot-password flows are implemented.

### Data Pipeline

Five independent pipelines with separate manifests: OSM extraction, border detection, Valhalla routing, Pelias geocoding, and vector tile generation. GTFS transit stop importer and Overture Maps places/addresses importer are included. Pipeline supports incremental re-runs and parallel execution.

## Dependencies

- None (starting milestone)

## Acceptance Criteria

- [x] All six backend services build and run
- [x] Map tiles render with city names, borders, and road labels
- [x] TilesService serves map styles from filesystem
- [x] MailService and RegionService operate without MinIO
- [x] SearchService provides geocoding via Pelias fan-out
- [x] Five independent data pipelines are runnable separately
- [x] Android app displays map with location search
- [x] Dark theme, remember-me, and forgot-password implemented
- [x] GTFS and Overture Maps importers integrated into pipeline
