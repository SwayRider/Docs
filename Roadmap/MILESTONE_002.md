# Milestone 002 - Route Planning

**Status**: In Progress

## Goals

Deliver end-to-end motorcycle route planning in the Android app. The user can set an origin and destination, configure route preferences specific to motorcycle riding (scenic vs. highway, toll avoidance, road surface preference), and receive a calculated route displayed on the map. The RouterService backend handles multi-region routing via Valhalla with seamless border crossing.

## Key Deliverables

### RouterService — Route Calculation API

RouterService exposes a gRPC/REST endpoint that accepts an origin, destination, and motorcycle-specific route preferences. It fans out requests across Valhalla regions as needed, handles border crossings, and returns a route with geometry, distance, duration, and turn-by-turn instructions.

### Route Planning UI (Android)

The Android app provides a route planning screen where the user can:
- Enter or pick origin and destination (via search or map tap)
- Select route preferences (scenic vs. highway, toll avoidance, unpaved road handling)
- Trigger route calculation
- View the resulting route drawn on the map with distance and estimated time

### Route on Map

The calculated route geometry is rendered as a polyline on the MapLibre map. The map viewport adjusts to fit the full route. Alternative routes (if supported) are shown as secondary lines.

### Route Summary Panel

A bottom sheet or overlay displays route summary: total distance, estimated ride time, major waypoints, and selected preferences. The user can confirm the route to proceed to navigation or discard and re-plan.

## Dependencies

- MILESTONE_001 (Foundation Complete) — map rendering, auth, search, and backend service infrastructure must be in place

## Acceptance Criteria

- [ ] RouterService calculates routes across Western Europe coverage area
- [ ] Route respects motorcycle preferences (scenic, highway, toll, surface)
- [ ] Border crossing between covered regions works seamlessly
- [ ] Route geometry renders as polyline on the Android map
- [ ] Map viewport fits the full route after calculation
- [ ] Route summary (distance, duration) is displayed to the user
- [ ] Origin and destination can be set via search or map tap
- [ ] Route planning works end-to-end on a physical Android device
