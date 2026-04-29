# SwayRider — Project Description

## Vision

SwayRider is a **commercial mobile motorcycle routing application** built on a fully in-house developed backend. The app targets recreational and touring motorcycle riders across Europe, offering route planning, turn-by-turn navigation, community features, and group riding capabilities.

The platform is delivered as a **subscription-based service** with a free tier offering limited functionality and one or more paid plans unlocking premium features.

## Current State

SwayRider is a mature monorepo containing:

- **6 Go backend microservices** communicating over gRPC
- **Android mobile application** (Kotlin, Jetpack Compose, Clean Architecture)
- **Python data pipeline** for processing OpenStreetMap data into vector tiles, routing graphs, and geocoding indices

Current geographic coverage is focused on **Western Europe**: Belgium, Netherlands, Luxembourg, France, Germany, and the Iberian Peninsula.

## End State

### Client Applications

| Platform | Technology | Status |
|----------|-----------|--------|
| Android | Kotlin, Jetpack Compose | In development |
| iOS | Kotlin Multiplatform (KMP) | Planned |

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

The data pipeline and regional routing architecture are designed to support incremental geographic expansion by adding new Valhalla/Pelias regions.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Mobile Clients                        │
│         Android (Compose)  ·  iOS (KMP)                 │
└──────────────────────────┬──────────────────────────────┘
                           │ REST / gRPC
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
│   queries    │ - Pelias fan │ - MBTiles (MVT)           │
│ - Borders    │   out        │ - Multi-zoom hierarchy    │
└──────────────┴──────────────┴───────────────────────────┘
                           │ gRPC
┌──────────────────────────▼──────────────────────────────┐
│                   Data Layer                             │
├─────────────────┬───────────────────────────────────────┤
│  PostgreSQL     │  Geodata (filesystem)                 │
│  - Users        │  - Valhalla routing tiles             │
│  - Tokens       │  - Pelias geocoding data              │
│  - JWT keys     │  - Region contours & border crossings │
└─────────────────┴───────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              Data Pipeline (Python)                      │
│  OSM processing  ·  Vector tile generation              │
│  Valhalla graph  ·  Pelias import  ·  Border detection  │
└─────────────────────────────────────────────────────────┘
```

## Deployment

| Phase | Model |
|-------|-------|
| MVP | Self-hosted infrastructure |
| Production | Cloud hosting or colocation (provider TBD) |

The Docker Compose layered architecture (layer-00 base, layer-10 geospatial, layer-20 SwayRider) supports both self-hosted and cloud deployment.

## Key Design Decisions

- **Custom vector tiles**: Map data is generated in-house from OpenStreetMap using a Python pipeline (Tippecanoe, Osmium). This provides full control over map styling, feature selection, and data freshness.
- **Regional routing**: Routes are calculated per-region with seamless border crossing handling, enabling horizontal scaling across geographies.
- **gRPC-first**: All inter-service communication uses gRPC with Protocol Buffers. External APIs are exposed via gRPC-gateway.
- **Multiplatform mobile**: Kotlin Multiplatform (KMP) enables shared business logic between Android and iOS while maintaining native UI on each platform.
