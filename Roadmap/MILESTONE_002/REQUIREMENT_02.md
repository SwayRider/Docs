# Requirement 02 - Route Calculation Backend

**Status**: Closed

**Scope**:
  - `backend/services/routerservice/`
  - `backend/protos/router/v1/`

## Goal

Harden the RouterService route calculation endpoint for MVP: ensure the proto contract exposes motorcycle-specific route preferences, that error handling is robust, and that multi-region border crossing works end-to-end across the Western Europe coverage area.

## Description

The RouterService already implements multi-region Valhalla routing with border-crossing logic. This requirement ensures the API contract and implementation are MVP-ready:

**Proto contract**: The `Route` request must include motorcycle-specific preference fields:
- `scenic_preference`: prefer scenic / winding roads over direct routes
- `highway_avoidance`: avoid motorways / highways
- `toll_avoidance`: avoid toll roads
- `unpaved_handling`: enum — prefer / neutral / avoid unpaved surfaces

**Response contract**: The `Route` response must include:
- Encoded route geometry (polyline or GeoJSON)
- Total distance in meters
- Total duration in seconds
- A human-readable summary (start region, end region)

**Error handling**: Clearly defined gRPC status codes for:
- No route found between origin and destination
- One or more coordinates outside covered regions
- Valhalla backend unreachable

**Multi-region coverage**: The routing logic correctly identifies when a route crosses regional Valhalla instance boundaries and sequences the sub-routes, producing a single merged response.

Note: Parallelization of multi-region requests is deferred to POST_MVP (see `POST_MVP.md`).

## Tasks

- [TASK_01](./REQUIREMENT_02/TASK_01.md) — Extend Proto + Implement Valhalla Costing Mapping
- [TASK_02](./REQUIREMENT_02/TASK_02.md) — Improve Error Handling with Proper gRPC Status Codes
- [TASK_03](./REQUIREMENT_02/TASK_03.md) — Comprehensive Error Case Tests + Multi-Region Verification

## Dependencies

- REQUIREMENT_01 (RouterService Authentication & Security) — endpoint must be secured before hardening

## Acceptance Criteria

- [ ] `Route` request proto includes all four motorcycle preference fields
- [ ] `Route` response proto includes geometry, distance (meters), duration (seconds), and summary
- [ ] A request with scenic preference set returns a route preferring winding roads (verified via Valhalla costing config)
- [ ] A route crossing at least two regional Valhalla instances is calculated and returned as a single merged geometry
- [ ] A request with origin or destination outside all covered regions returns a well-formed gRPC error (`NOT_FOUND` or `INVALID_ARGUMENT`)
- [ ] A request when a Valhalla backend is unreachable returns a well-formed gRPC error (`UNAVAILABLE`)
- [ ] The service passes all existing unit tests and new tests covering the above error cases
