# TASK_001 — Backend: RouteType Enum

**Status**: Planned

## Overview

Add a `RouteType` enum to the RouterService proto and implement preset preference mapping on the backend. Multi-region routing, border crossing, and individual avoidance preferences are already implemented — this task only adds the named route type concept.

## Repository

- **Repo**: swayrider
- **Subfolder**: `backend/`
- **Tech**: Go, gRPC, Protocol Buffers

## Background

The backend already supports:
- Motorcycle routing (`RM_MOTORCYCLE` costing model)
- Multi-region routing with border crossing
- Preference floats: `highway_preference`, `tollway_preference`, `ferry_preference`, `trail_preference`, `shortest_path`, `use_distance`

What's missing: a first-class `RouteType` that maps to a preset combination of these preferences, so the mobile app doesn't need to know the internal preference values.

## Technical Specification

### Proto Changes (`backend/protos/router/v1/router.proto`)

Add enum:
```protobuf
enum RouteType {
  RT_UNSPECIFIED = 0;  // defaults to RT_FASTEST
  RT_FASTEST = 1;      // minimize travel time
  RT_SHORTEST = 2;     // minimize distance
  RT_SCENIC = 3;       // prefer scenic/rural roads, avoid highways
}
```

Add field to `RouteRequest`:
```protobuf
RouteType route_type = <next_field_number>;
```

### Backend Mapping (`backend/services/routerservice/internal/logic/routing_request.go`)

In the function that builds `RouteOptions`, apply preset preferences based on `route_type` **before** applying any explicit `route_options` overrides (so the caller can still override individual values):

| RouteType | shortest | use_distance | highway_preference | trail_preference |
|-----------|----------|--------------|-------------------|-----------------|
| RT_FASTEST / unspecified | false | 0.0 | 1.0 (default) | 0.5 (default) |
| RT_SHORTEST | true | 1.0 | — | — |
| RT_SCENIC | false | 0.0 | 0.1 | 0.9 |

RT_SCENIC also sets `tollway_preference = 0.2` by default.

### Proto Regeneration

After changing the proto:
```bash
cd backend/protos && make
```

## Dependencies

- None (backend-only change)

## Acceptance Criteria

- [ ] `RouteType` enum exists in proto with RT_FASTEST, RT_SHORTEST, RT_SCENIC
- [ ] `route_type` field added to `RouteRequest`
- [ ] Backend maps RT_SCENIC to high trail preference + low highway preference
- [ ] Backend maps RT_SHORTEST to shortest=true + use_distance=1.0
- [ ] Explicit `route_options` fields still override the preset when provided
- [ ] Proto regenerated and backend compiles

## Testing Notes

Use Bruno collection in `rest/` to call the Route endpoint with `route_type: 3` (scenic) and verify the Valhalla costing options in the log include low highway preference.
