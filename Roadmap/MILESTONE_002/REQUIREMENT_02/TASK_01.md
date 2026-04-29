# Task 01 - Extend Proto + Implement Valhalla Costing Mapping

**Status**: closed

## Scope

- `backend/protos/router/v1/`
- `backend/services/routerservice/internal/server/`
- `backend/services/routerservice/internal/logic/`

## Description

Extend the proto contract and implement the Valhalla costing mapping for motorcycle-specific route preferences:

1. **Proto extensions** - Add motorcycle preference fields to `RouteRequest`:
   - `scenic_preference` (float) — prefer scenic/winding roads over direct routes
   - `highway_avoidance` (float) — avoid motorways/highways
   - `toll_avoidance` (float) — avoid toll roads
   - `unpaved_handling` (enum) — prefer / neutral / avoid unpaved surfaces

2. **Proto extensions** - Add human-readable summary to `RouteResponse`:
   - Start region name (e.g., "France-North")
   - End region name (e.g., "Germany-West")
   - Computed from region assignments during multi-region routing

3. **Valhalla costing mapping** - Wire the new preference fields to Valhalla costing options in `createRequestOptions`:
   - `scenic_preference` → Valhalla `use_trails`, `use_ferry` with scenic-oriented values
   - `highway_avoidance` → Valhalla `use_highways` (inverse relationship)
   - `toll_avoidance` → Valhalla `use_tolls` (inverse relationship)
   - `unpaved_handling` → Valhalla `exclude_unpaved` or `use_tracks` based on enum value

These should work similarly to how `RouteType_RT_SCENIC` already sets presets.

## Instructions

### Proto Changes (`backend/protos/router/v1/router.proto`)

1. Add `UnpavedHandling` enum with values: `UH_PREFER`, `UH_NEUTRAL`, `UH_AVOID`
2. Add motorcycle preference fields to `RouteRequest`:
   ```protobuf
   optional float scenic_preference = 13;
   optional float highway_avoidance = 14;
   optional float toll_avoidance = 15;
   optional UnpavedHandling unpaved_handling = 16;
   ```
3. Add `RouteSummary` message with `start_region` and `end_region` fields
4. Add `RouteSummary` to `RouteResponse`

### Implementation Changes

**File: `backend/services/routerservice/internal/server/route.go`**

- Extend `createRequestOptions` to handle the four new preference fields
- Modify `buildCombinedRouteResponse` to compute and populate the human-readable summary from region assignments
- Pass region information through the response building chain

**File: `backend/services/routerservice/internal/logic/routing_request.go`**

- Add new `ScenicPreferenceOption`, `HighwayAvoidanceOption`, `TollAvoidanceOption`, `UnpavedHandlingOption` functions if needed

### Regenerate Proto

- Run protobuf code generation to produce updated Go code

## Test Scenarios

1. **Proto field parsing** — Verify all four preference fields are correctly parsed in `createRequestOptions`
2. **Valhalla costing output** — Verify the new fields produce correct Valhalla costing values
3. **Scenic preference** — With scenic_preference set, verify `use_trails` and `use_ferry` are set appropriately
4. **Highway avoidance** — With highway_avoidance=0.9, verify `use_highways` is low (inverse)
5. **Toll avoidance** — With toll_avoidance=0.8, verify `use_tolls` is low (inverse)
6. **Unpaved handling** — With each enum value, verify correct Valhalla options
7. **Summary generation** — Verify start/end region names are populated in response
8. **Override behavior** — Explicit preferences should override RouteType presets

## Acceptance Criteria

- [ ] `RouteRequest` proto includes `scenic_preference`, `highway_avoidance`, `toll_avoidance`, `unpaved_handling` fields
- [ ] `RouteResponse` proto includes human-readable summary with start/end region names
- [ ] `createRequestOptions` correctly maps all four preference fields to Valhalla costing options
- [ ] Scenic preference produces winding-road routes via Valhalla costing config
- [ ] All existing unit tests continue to pass
- [ ] New unit tests for preference field mapping pass
- [ ] Service compiles without errors (`go build ./...` in `backend/`)
