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

## Data Pipeline

### Python

Geodata processing uses Python:

- **GeoPandas/Shapely**: Rich spatial data manipulation
- **Osmium**: Fast OSM PBF parsing and extraction
- **Ecosystem**: Tippecanoe, GDAL/ogr2ogr integrate naturally
- **Flexibility**: Rapid iteration on tile generation logic

### Tippecanoe

Vector tile generation from GeoJSON:

- **Control**: Fine-grained zoom-dependent simplification and feature dropping
- **Performance**: Optimized MBTiles output for MapLibre rendering
- **Tuning flags**: `--coalesce-densest-as-needed`, `--simplification`, `--buffer` for visual quality

### Five Independent Pipelines

Data generation is split into independent pipelines with separate manifests:

| Pipeline | Output | Dependency |
|----------|--------|------------|
| OSM extraction | Regional `.osm.pbf` files | None |
| Border detection | Region contours, border crossings | OSM |
| Valhalla routing | Routing graph tiles | OSM |
| Pelias geocoding | Geocoding index | OSM |
| Tiles | MBTiles vector tiles | None (independent) |

This allows partial re-runs and parallel execution where dependencies permit.

## Map Rendering

### Custom Vector Tiles (MapLibre)

Maps are rendered from in-house vector tiles using MapLibre:

- **Full control**: Feature selection, simplification, styling all managed internally
- **Cost**: No per-request tile fees (vs Mapbox, Google Maps)
- **Offline support**: MBTiles files can be bundled or downloaded for offline use
- **Customization**: Motorcycle-specific styling (road surface, scenic highlights, fuel stops)
- **Zoom hierarchy**: L0 (world) → L1 (continental) → L2 (regional) → L3 (local) for progressive detail

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
| Minio (object storage) | Removed | Mail templates moved to database-backed storage |
| Kotlin/Jetpack Compose Android prototype | Removed | Replaced by the Flutter app (single iOS + Android codebase) |

## Decisions Pending

| Topic | Status |
|-------|--------|
| iOS release timeline | TBD — Flutter covers both platforms; iOS release follows Android MVP |
| Cloud provider selection | TBD — MVP self-hosted; production cloud/colo evaluated at scale |
| Subscription payment integration | TBD — Payment provider not yet selected |
| Offline map download strategy | TBD — Tile bundling vs on-demand download |
| POI data sources | TBD — OSM amenity tags vs third-party data |
