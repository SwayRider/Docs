# TASK_004 — Mobile: Route Planning UI

**Status**: Planned

## Overview

Replace the `RoutePlanningScreen` placeholder with a functional route planning UI: origin/destination selection, route type and avoidance preferences, calculate action, and route summary display.

## Repository

- **Repo**: swayrider
- **Subfolder**: `mobile/android/`
- **Tech**: Kotlin, Jetpack Compose

## Background

`RoutePlanningScreen.kt` currently shows placeholder text. The existing `LocationSearchBar` component can be reused for origin/destination input. The screen is already registered in the navigation graph.

## Technical Specification

### Screen Layout

```
┌─────────────────────────────┐
│  ← Route Planning           │  (AppScaffold header, existing)
├─────────────────────────────┤
│  [📍] Origin          [GPS] │  (LocationSearchBar reuse)
│  [🏁] Destination           │  (LocationSearchBar reuse)
│       [⇅ Swap]              │
├─────────────────────────────┤
│  Route type:                │
│  [Fastest] [Shortest] [Scenic] │  (filter chips, single select)
├─────────────────────────────┤
│  Avoid:                     │
│  [Highways ☐] [Tolls ☐] [Ferries ☐]  │  (toggles/checkboxes)
├─────────────────────────────┤
│  [     Calculate Route     ]│  (enabled when origin+dest set)
├─────────────────────────────┤
│  ── Route Summary ──        │  (visible when calculationState = Success)
│  🕐 1h 23min   📏 87 km    │
│  [  View on Map  ]          │  (navigates back to HomeScreen with route)
└─────────────────────────────┘
```

### Implementation Notes

- Reuse `LocationSearchBar` for origin and destination (it has autocomplete built in)
- "GPS" button on origin field: calls `viewModel.setOriginFromGps()` using last known location from `HomeScreen`'s location state — pass current lat/lng as a nav argument or via a shared ViewModel
- Route type chips: use `FilterChip` from Material 3
- Avoid toggles: use `Switch` or `Checkbox` from Material 3
- Calculate button: disabled (greyed out) when origin or destination is null
- Loading state: show `CircularProgressIndicator` instead of button label
- Error state: show a `Snackbar` with the error message
- Route summary card: visible only when `calculationState is RouteCalculationState.Success`
- "View on Map" button: navigate to HomeScreen passing the route result (encoded polyline + distance/duration) via nav arguments or shared state

### Navigation

The "View on Map" action should pass the route result to HomeScreen so the polyline can be drawn (implemented in TASK_005). Use a shared ViewModel at the activity scope, or nav arguments for the encoded polyline string.

## Dependencies

- TASK_003 (RoutePlanningViewModel must exist)

## Acceptance Criteria

- [ ] Origin and destination fields work with autocomplete search
- [ ] GPS button sets origin to current location with label "My Location"
- [ ] Swap button exchanges origin and destination
- [ ] Route type chips (Fastest/Shortest/Scenic) are single-select and default to Fastest
- [ ] Avoid toggles (Highways, Tolls, Ferries) update preferences
- [ ] Calculate button is disabled when origin or destination is missing
- [ ] Loading indicator shown during calculation
- [ ] Error message shown on failure
- [ ] Route summary (distance, time) shown on success
- [ ] "View on Map" navigates to HomeScreen with route data

## Testing Notes

Manual test: enter an origin and destination, select Scenic, toggle Avoid Highways, calculate. Verify summary shows reasonable distance/time values for the Belgian/Dutch/German region.
