# TASK_005 — Mobile: Route Display on Map

**Status**: Planned

## Overview

Draw the calculated route as a polyline on the MapLibre map in `HomeScreen`, with origin and destination markers. The route is passed from the route planning flow.

## Repository

- **Repo**: swayrider
- **Subfolder**: `mobile/android/`
- **Tech**: Kotlin, Jetpack Compose, MapLibre

## Background

`HomeScreen.kt` already integrates MapLibre with location tracking, GPS marker, and search result markers. Route polyline drawing requires adding a GeoJSON source + line layer to the existing map setup. MapLibre supports this natively.

## Technical Specification

### Route Data Flow

A shared or passed route result from `RoutePlanningViewModel` needs to reach `HomeScreen`. Recommended: use a shared ViewModel at the activity scope (or NavBackStackEntry scope for the home destination) so both screens can read/write the active route.

```kotlin
// Shared state: active route (null when no route calculated)
val activeRoute: StateFlow<RouteResult?> = ...
```

### Map Layer Setup (inside `AndroidView` / `MapView` callback)

When `activeRoute` changes in `HomeScreen`:

1. **Clear previous route** — remove existing GeoJSON source and line layer if present
2. **Decode polyline** — convert `RouteResult.encodedPolyline` to a list of `LatLng` coordinates
3. **Add GeoJSON source** — create a LineString feature from the coordinates
4. **Add line layer** — style: color `#5B8DEF`, width 5dp, cap ROUND, join ROUND
5. **Add markers** — origin marker (green pin) and destination marker (red pin) using MapLibre symbol layer or annotation API
6. **Fit camera** — call `easeCamera(CameraUpdateFactory.newLatLngBounds(...))` to show the full route with padding

### Encoded Polyline Decoding

The RouterService returns Valhalla's encoded polyline format (precision 6). Use a decoder utility or implement inline:
- Standard Google polyline algorithm with 1e-6 precision (Valhalla default)
- Result is `List<LatLng>`

### UI Integration in HomeScreen

Add a "Clear route" FAB or button visible when `activeRoute != null`. Tapping it clears the shared route state and removes the map layers.

Optionally: show a persistent bottom sheet summary bar (distance + time) when a route is active.

## Dependencies

- TASK_003 (RouteResult domain model must exist)
- TASK_004 (route is passed from RoutePlanningScreen via shared state)

## Acceptance Criteria

- [ ] Route polyline drawn on map after returning from RoutePlanningScreen
- [ ] Polyline styled with correct color and width
- [ ] Origin marker (green) and destination marker (red) shown
- [ ] Camera fits the full route on screen
- [ ] Previous route cleared when a new one is calculated
- [ ] "Clear route" action removes polyline and markers
- [ ] Route summary bar (distance, time) visible in HomeScreen while route is active

## Testing Notes

Calculate a route between two Belgian cities. Verify the polyline matches the expected road geometry and both markers appear at the correct positions.
