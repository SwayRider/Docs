# Requirement 05 - Route Rendering on Map

**Status**: Open

**Scope**:
  - `mobile/android/` — map rendering layer, RouterService API client

## Goal

After a route is calculated, render the route geometry as a styled polyline on the MapLibre map and adjust the map viewport to fit the full route within view.

## Description

When the route calculation response is received from RouterService:

### Polyline Rendering

- The route geometry (encoded polyline or GeoJSON from the RouterService response) is decoded and drawn as a MapLibre line layer on the map.
- The line is styled to be clearly visible on the map: a distinct color (e.g. accent blue), sufficient stroke width, and a subtle outline/halo for contrast against different tile backgrounds.
- Any previously rendered route is cleared before drawing the new one.

### Viewport Fit

- After drawing the polyline, the map camera animates to fit the full route within the visible viewport with appropriate padding on all sides (accounting for the Route Planning input bar at the top and the Route Summary Panel at the bottom — see REQUIREMENT_06).

### Origin and Destination Markers

- A start marker (pin or circle) is placed at the route origin coordinate.
- An end marker is placed at the route destination coordinate.
- Markers are styled distinctly (e.g. green for start, red for end).

### RouterService API Client (Android)

- A `RouterServiceApi` Retrofit (or gRPC) client is added to the Android DI setup, following the same `@AuthRequired` JWT injection pattern as `SearchServiceApi`.
- The client is wired in `SwayRiderApp.kt`.

## Tasks

_Tasks to be defined._

## Dependencies

- REQUIREMENT_01 (RouterService Authentication & Security) — JWT-secured endpoint must be in place
- REQUIREMENT_02 (Route Calculation Backend) — response contract (geometry, distance, duration) must be defined
- REQUIREMENT_04 (Route Planning Screen) — route calculation is triggered from this screen

## Acceptance Criteria

- [ ] Calling `RouterServiceApi.calculateRoute(...)` from the Android app reaches the secured RouterService endpoint with a valid JWT
- [ ] The route polyline is drawn on the MapLibre map after a successful response
- [ ] Any previously drawn route is removed before drawing a new one
- [ ] The map viewport animates to fit the full route with appropriate padding
- [ ] A start marker and end marker are visible at the route endpoints
- [ ] The polyline is clearly visible against both light and dark map tile backgrounds
- [ ] Route rendering works on a physical Android device
