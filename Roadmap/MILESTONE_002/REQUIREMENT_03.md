# Requirement 03 - Map Interaction & Waypoint Selection

**Status**: Open

**Scope**:
  - `mobile/android/` — map screen, map interaction layer

## Goal

Allow the user to select route waypoints directly from the map via two interaction patterns: tapping a search result marker and long-pressing anywhere on the map. Both interactions lead to an action sheet that lets the user set the tapped location as the route origin or destination.

## Description

Two map interaction patterns must be implemented:

### Search Result Marker Tap

When the user performs a search from the home screen and a result marker is shown on the map, tapping the marker opens an action sheet with the following options:
- "Navigate to" — sets this location as the destination and opens the Route Planning Screen with origin pre-filled as current location
- "Set as destination" — sets this location as the destination in the Route Planning Screen
- "Set as start point" — sets this location as the origin in the Route Planning Screen

### Long-Press Reverse Geocode

When the user long-presses anywhere on the map:
1. A dropped pin is placed at the pressed coordinate.
2. A reverse geocode request is made to SearchService to resolve the coordinate to a human-readable address.
3. A loading indicator is shown on the pin while the address is being resolved.
4. Once resolved, the pin label shows the address. If reverse geocoding fails, the pin label shows the coordinate (lat, lon) formatted to 5 decimal places.
5. The same action sheet as above is shown: "Navigate to", "Set as destination", "Set as start point".

The dropped pin is dismissed when the user taps elsewhere on the map or cancels the action sheet.

## Tasks

| File | Title | Status |
| ---- | ----- | ------ |
| REQUIREMENT_03/TASK_01.md | ReverseGeocode RPC: Proto & SearchService Backend | open |
| REQUIREMENT_03/TASK_02.md | SearchServiceApi: ReverseGeocode Mobile Client | open |
| REQUIREMENT_03/TASK_03.md | SelectedLocation Domain Class & RoutePlanningViewModel | open |
| REQUIREMENT_03/TASK_04.md | Search Result Marker Tap → Action Sheet | open |
| REQUIREMENT_03/TASK_05.md | Long-Press Pin Drop & Reverse Geocode Flow | open |

## Dependencies

- MILESTONE_001 (Foundation Complete) — MapLibre map rendering and SearchService integration must be in place
- REQUIREMENT_01 (RouterService Authentication & Security) — JWT auth in place so the Route Planning Screen can make authenticated calls

## Acceptance Criteria

- [ ] Tapping a search result marker on the map opens an action sheet with "Navigate to", "Set as destination", "Set as start point"
- [ ] Each action sheet option correctly pre-fills the Route Planning Screen with the selected location
- [ ] Long-pressing the map places a dropped pin at the pressed coordinate
- [ ] A reverse geocode call is made to SearchService for the long-pressed coordinate
- [ ] The pin displays the resolved address once the call completes
- [ ] If reverse geocoding fails, the pin displays the coordinate formatted as lat/lon
- [ ] The dropped pin action sheet offers "Navigate to", "Set as destination", "Set as start point"
- [ ] Tapping elsewhere on the map dismisses the dropped pin
- [ ] The interaction works on a physical Android device
