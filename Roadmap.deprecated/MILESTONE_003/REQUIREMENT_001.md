# REQUIREMENT_001 — Route Planning

## Overview

Implement motorcycle route planning with customizable preferences, leveraging the existing RouterService and Valhalla routing engine.

## Context

- **Components**: RouterService, Mobile App
- **Priority**: Critical
- **Status**: Planned

## Requirements

### Route Calculation
- Motorcycle routing
- Support for different route types:
  - Fastest route
  - Shortest route
  - Scenic route (avoid highways, prefer scenic roads)
  - Custom preferences (toll avoidance, highway avoidance, etc.)
- Multi-region routing with seamless border crossing
- Route recalculation on deviation

### Mobile App UI
- Origin and destination selection (from map, search, or GPS)
- Route preference selection
- Multiple route options displayed on map
- Route summary (distance, time, waypoints)
- Save/favorite routes

### Route Preferences
| Preference | Options |
|------------|---------|
| Route Type | Fastest, Shortest, Scenic |
| Avoid Highways | Yes/No |
| Avoid Tolls | Yes/No |
| Avoid Ferries | Yes/No |
| Motorcycle Type | Sport, Touring, Cruiser (affects routing) |

## Acceptance Criteria

1. Routes calculate correctly across European regions
2. Border crossing handled seamlessly
3. Route preferences affect routing output
4. Multiple route options displayed
5. Route summary shows accurate distance and time
6. Routes can be saved for later use
7. Recalculation works on deviation

## Affected Files

### Backend
- `backend/services/routerservice/` — Route calculation logic
- `backend/protos/routerservice/` — Proto definitions

### Mobile App
- `mobile/android/app/src/main/java/.../ui/routing/` — Route planning UI
- `mobile/android/app/src/main/java/.../data/routing/` — Route data layer
