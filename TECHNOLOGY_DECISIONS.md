# SwayRider — Technology Decisions

## Backend

### Go

All backend services are written in Go. Key reasons:

- **Performance**: Low-latency gRPC services with minimal memory overhead
- **Concurrency**: Native goroutines for handling parallel Valhalla/Pelias requests
- **Static typing**: Compile-time safety for a large monorepo
- **Deployment**: Single binary deployment, fast container builds
- **Ecosystem**: Mature gRPC, Protocol Buffer, and PostgreSQL libraries

### gRPC + Protocol Buffers

Inter-service communication exclusively uses gRPC with Protocol Buffers:

- **Type safety**: Schema-defined contracts between services
- **Performance**: Binary serialization, HTTP/2 multiplexing
- **Code generation**: Auto-generated Go clients from `.proto` definitions
- **Gateway**: gRPC-gateway exposes REST APIs for mobile clients without duplicating logic
- **Streaming**: Available for future real-time features (group riding, navigation)

### PostgreSQL

Primary database for auth and user data:

- **Reliability**: ACID compliance for user accounts, tokens, JWT keys
- **Migrations**: Managed via `sql-migrate`
- **Mature tooling**: Extensive Go driver support (`lib/pq`)

## Mobile

### Flutter (Dart)

The mobile client (the earlier Kotlin/Jetpack Compose prototype) was replaced by a single Flutter app covering iOS and Android:

- **Single codebase**: One Dart codebase for both platforms — UI, business logic, and API integration are shared; no per-platform UI layer to maintain
- **Declarative UI**: Widget-based reactive UI with Material 3 theming and a shared design system (color palette, theme, responsive dimension sets)
- **Team efficiency**: One language and one codebase to maintain instead of separate Android/iOS implementations
- **State management**: Provider-based DI with an MVVM + Command/Result pattern — view models expose `Command` objects and sealed `Result<T>` types for actions
- **Tooling**: `go_router` for navigation, freezed + json_serializable for models, `shared_preferences` for token persistence

### Clean Architecture (MVVM)

Mobile app follows a layered architecture (based on the Flutter "Compass App" sample):

```
UI (widgets) → ViewModel → Repository → API client / storage
```

- **Testability**: View models and repositories are framework-agnostic, easily unit tested
- **Separation of concerns**: UI never calls the API client directly; repositories mediate all data access
- **Manual DI**: Provider wiring in the app entry point; no code-gen DI framework

## Data Manager

> The Python `data-pipeline/` (CLI scripts, Tippecanoe/MBTiles tiles, five independent pipelines with tarball distribution) is **deprecated** and replaced by `data-manager/`. Steps and deployment strategy: [MIGRATION-DATA-MANAGER.md](./MIGRATION-DATA-MANAGER.md).

### Flask + RQ + SQLite

`data-manager` is a Flask UI (server-rendered Jinja + htmx) with an RQ worker. Flask never runs domain logic; it reads/writes SQLite (WAL, SQLAlchemy + Alembic) and enqueues jobs. Long builds run in the worker; configuration is authored in a map UI (countries → regions → automatic overlap and border pairs) rather than YAML.

- **GeoPandas/Shapely/pyogrio**: spatial work (polygons, overlap, borders)
- **Osmium/GDAL**: planet extraction per country in one pass, region extracts
- **Docker**: temporary per-region Elasticsearch for Pelias imports

### Stages with typed assets

Each stage declares the asset types it `produces` and `consumes`; order is resolved by topological sort. A run ends in `awaiting_review`, and only **approved** assets feed later stages.

| Stage family | Output |
|--------------|--------|
| OSM (`download-planet`, `extract-countries`, `osm-extract`) | Per-country and per-region `.osm.pbf` |
| Border | Region outlines (core/extended), border-crossing CSVs |
| Valhalla | Routing tiles, admin/timezone sqlite, polylines per region |
| Pelias | ES index snapshot, config, patched WOF, interpolation DBs, transit (GTFS), Overture/OpenAddresses data |
| Tiles/styles | Protomaps planet PMTiles (download), styles, glyphs, sprites |

### Release + copy deployment

Output is published as immutable releases and **copied** (rsync over ssh, checksum-verified) to the target server, which switches a per-artifact `current` symlink. This lets the data-manager run on a separate build host and artifact classes live on separate drives. Publish/deploy is planned (migration Phase A4).

## Map Rendering

### Custom Vector Tiles (MapLibre)

Maps are rendered client-side with MapLibre from self-served vector tiles:

- **Tiles**: Protomaps planet **PMTiles v3** (gzip MVT, z0–15, client over-zoom), downloaded by `data-manager`, stored in an S3-compatible object store (Garage, self-hosted; decision 2026-10-05, reversing the earlier "no object store") and read by `tilesservice` with ranged GETs (a local-file backend remains for tests and laptops). Replaces the in-house Tippecanoe/MBTiles L0–L3 hierarchy (legacy `base` tileset is served side-by-side until cutover).
- **Styles, glyphs, sprites**: Shipped in the same release and served by `tilesservice`; clients never read the file directly.
- **Full control**: Styling is managed internally (motorcycle-specific styling: road surface, scenic highlights, fuel stops)
- **Cost**: No per-request tile fees (vs Mapbox, Google Maps)
- **Offline support**: Strategy pending (see KEY_DECISIONS); PMTiles supports range-based regional extracts

Alternatives considered:
- **Mapbox**: Per-request pricing incompatible with offline-first strategy
- **Google Maps**: Limited customization, no offline vector tiles, high cost

## Security

### JWT (RS256) with Key Rotation

- **Asymmetric signing**: RS256 allows public key distribution for verification without exposing private keys
- **Key rotation**: Automatic hourly key checks, seamless verification across rotation boundaries
- **Short-lived tokens**: Access tokens paired with single-use refresh tokens

### Argon2id Password Hashing

- **Memory-hard**: Resistant to GPU/ASIC attacks
- **OWASP recommended**: Current best practice for password storage
- **Centralized in swlib**: All services use the same hashing implementation

### Service-to-Service Authentication

- **Service clients**: Dedicated credentials for inter-service calls
- **Scope-based access**: Endpoints explicitly declare required security level (Public, Unverified, Admin, ServiceClient)
- **gRPC interceptors**: Security enforcement at transport layer, not application layer

## Infrastructure

### Docker Compose (Layered)

Infrastructure is organized in dependency layers:

```
layer-00 (Base):        Traefik, PostgreSQL, Elasticsearch, Redis, WireGuard
layer-10 (Geospatial):  Valhalla (per-region), Pelias (per-region)
layer-20 (SwayRider):   Auth, Mail, Region, Router, Search, Tiles services
layer-30 (Web):         swayrider-api (API gateway)
```

Each layer builds on the previous. This supports:
- **Local development**: Start only the layers you need
- **Incremental deployment**: Deploy infrastructure in dependency order
- **Isolation**: Geospatial services (resource-intensive) separated from application services

### Container Registry

Service containers are pushed to GitHub Container Registry (`ghcr.io/swayrider`) via multi-platform builds (linux/amd64, linux/arm64).

## Removed / Deprecated Technologies

| Technology | Status | Reason |
|------------|--------|--------|
| React web auth portal | Deprecated | Mobile-first strategy; web removed from scope |
| Minio (object storage) | Removed | Mail templates moved to database-backed storage. (Object storage returns for the planet tiles, as Garage; see Tiles above.) |
| Kotlin/Jetpack Compose Android prototype | Removed | Replaced by the Flutter app (single iOS + Android codebase) |

## Decisions Pending

| Topic | Status |
|-------|--------|
| iOS release timeline | TBD — Flutter covers both platforms; iOS release follows Android MVP |
| Cloud provider selection | TBD — MVP self-hosted; production cloud/colo evaluated at scale |
| Subscription payment integration | TBD — Payment provider not yet selected |
| Offline map download strategy | TBD — Tile bundling vs on-demand download |
| POI data sources | TBD — OSM amenity tags vs third-party data |
