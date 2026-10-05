# SwayRider — Project Description

## Vision

SwayRider is a **commercial mobile motorcycle routing application** built on a fully in-house developed backend. The app targets recreational and touring motorcycle riders across Europe, offering route planning, turn-by-turn navigation, community features, and group riding capabilities.

The platform is delivered as a **subscription-based service** with a free tier offering limited functionality and one or more paid plans unlocking premium features.

## Current State

SwayRider is a mature monorepo containing:

- **7 Go backend services** — an API gateway (`swayrider-api`) plus six microservices communicating over gRPC
- **Flutter mobile application** — a single Dart codebase covering both iOS and Android
- **Python data-manager** (Flask + RQ; replaces the deprecated `data-pipeline`) for processing OpenStreetMap data into routing graphs, geocoding indices (incl. address interpolation and transit) and border data, and for fetching planet vector tiles

Current geographic coverage is focused on **Western Europe**: Belgium, Netherlands, Luxembourg, France, Germany, and the Iberian Peninsula.

## End State

### Client Applications

| Platform | Technology | Status |
|----------|-----------|--------|
| iOS + Android | Flutter (Dart) | In development |

The mobile apps are the primary user-facing interface.

### Core Features

| Feature | Description |
|---------|-------------|
| **Route Planning** | Multi-modal motorcycle routing with customizable preferences (highway vs scenic, toll avoidance, etc.) |
| **Turn-by-Turn Navigation** | Real-time GPS navigation with voice guidance and lane assistance |
| **Offline Maps** | Downloadable vector map tiles for offline/low-connectivity use |
| **Geocoding & Search** | Address and place search with multi-region Pelias backend |
| **Points of Interest** | Restaurants, fuel stations, scenic viewpoints, rest areas, repair shops |
| **Route Sharing** | Community hub for sharing, rating, and discovering routes |
| **Group Riding** | Real-time group coordination: shared route, participant monitoring, stop/fuel requests |

### Subscription Model

| Tier | Capabilities |
|------|-------------|
| **Free** | Basic route planning, limited map downloads, standard search |
| **Paid (Tier 1)** | Full turn-by-turn navigation, unlimited offline maps, POI access |
| **Paid (Tier 2)** | Group riding features, route sharing/community, priority support |

Exact feature distribution across tiers is to be determined.

### Geographic Scope

| Phase | Coverage |
|-------|----------|
| MVP | Western Europe (current regions) |
| Phase 2 | Pan-European expansion (Central, Northern, Southern, Eastern Europe) |

The data-manager and regional routing architecture are designed to support incremental geographic expansion by adding new Valhalla/Pelias regions.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Mobile Clients                        │
│            Flutter (iOS · Android)                       │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS
┌──────────────────────────▼──────────────────────────────┐
│              swayrider-api  (API Gateway)                │
│  - JWT validation & rate limiting (Redis sliding window) │
│  - Redis Streams queue + SSE for routing & search        │
│  - HTTP reverse proxy for tiles and auth web pages       │
└──────────────────────────┬──────────────────────────────┘
                           │ gRPC
┌──────────────────────────▼──────────────────────────────┐
│                  Backend Services (Go)                   │
├──────────────┬──────────────┬───────────────────────────┤
│ AuthService  │ MailService  │ RouterService             │
│ - JWT auth   │ - SMTP email │ - Multi-region routing    │
│ - User mgmt  │ - Templates  │ - Valhalla integration    │
│ - Tokens     │              │ - Border crossing         │
├──────────────┼──────────────┼───────────────────────────┤
│ RegionService│ SearchService│ TilesService              │
│ - Spatial    │ - Geocoding  │ - Vector tile serving     │
│   queries    │ - Pelias fan │ - PMTiles (MVT) + styles  │
│ - Borders    │   out        │ - Fonts, sprites          │
└──────────────┴──────────────┴───────────────────────────┘
                           │ gRPC / SQL
┌──────────────────────────▼──────────────────────────────┐
│                   Data Layer                             │
├─────────────────┬──────────────┬────────────────────────┤
│  PostgreSQL     │  Redis       │  Geodata (filesystem)  │
│  - Users        │  - Job queue │  - Valhalla tiles      │
│  - Tokens       │  - Rate limit│  - Pelias data         │
│  - JWT keys     │    windows   │  - Region borders      │
└─────────────────┴──────────────┴────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              Data Manager (Python, build host)           │
│  OSM extraction  ·  Planet PMTiles + styles download    │
│  Valhalla graph  ·  Pelias + interpolation  ·  Borders  │
└──────────────────────────┬──────────────────────────────┘
                           │ release copy (rsync) + activate
                           ▼ Geodata (filesystem, per-artifact roots)
```

## Deployment

| Phase | Model |
|-------|-------|
| MVP | Self-hosted infrastructure |
| Production | Cloud hosting or colocation (provider TBD) |

The Docker Compose layered architecture (layer-00 base, layer-10 geospatial, layer-20 SwayRider) supports both self-hosted and cloud deployment.

## Key Design Decisions

- **Self-served vector tiles**: Planet PMTiles (Protomaps builds of OpenStreetMap) are fetched by the data-manager and served, with in-house styles, fonts and sprites, by `tilesservice`. This keeps full control over map styling and avoids per-request tile fees. (Previously tiles were built in-house with Tippecanoe; see [MIGRATION-DATA-MANAGER.md](./MIGRATION-DATA-MANAGER.md).)
- **Regional routing**: Routes are calculated per-region with seamless border crossing handling, enabling horizontal scaling across geographies.
- **gRPC-first**: All inter-service communication uses gRPC with Protocol Buffers. External APIs are exposed via gRPC-gateway.
- **Cross-platform mobile**: A single Flutter (Dart) codebase serves both iOS and Android, sharing UI, business logic, and API integration.
