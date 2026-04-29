# Requirement 06 - Route Summary Panel

**Status**: Open

**Scope**:
  - `mobile/android/` — route summary bottom sheet UI

## Goal

Display a route summary panel after a route is calculated, giving the user key information (distance and duration) and the ability to confirm the route (proceeding to navigation) or discard it and return to planning.

## Description

After a successful route calculation and map rendering (REQUIREMENT_05), a bottom sheet slides up from the bottom of the screen showing the route summary.

### Content

The panel displays:
- Total route distance (formatted: km with one decimal place, e.g. "142.3 km")
- Estimated ride duration (formatted: hours and minutes, e.g. "1 h 47 min")
- The selected route preferences as a compact summary line (e.g. "Scenic · No tolls")

### Actions

Two buttons are present:
- **"Start navigation"** (primary) — confirms the route and transitions to the navigation flow (MILESTONE_003). For MVP this button may show a "Coming soon" toast if turn-by-turn navigation is not yet available within MILESTONE_002.
- **"Discard"** (secondary) — removes the rendered route from the map and returns the user to the Route Planning Screen with the From/To fields preserved.

### Interaction

- The bottom sheet is non-dismissible by swipe (the user must use Discard or Start navigation to leave the state).
- The Route Planning Screen input bar remains visible above the bottom sheet (collapsed or partially visible) so the user can see what was planned.

## Tasks

_Tasks to be defined._

## Dependencies

- REQUIREMENT_02 (Route Calculation Backend) — distance and duration fields must be present in the response
- REQUIREMENT_05 (Route Rendering on Map) — the panel appears after route rendering is complete

## Acceptance Criteria

- [ ] A bottom sheet appears after a route is successfully calculated and rendered
- [ ] The panel shows total distance formatted as "X.X km"
- [ ] The panel shows estimated duration formatted as "X h YY min" (or "YY min" if under one hour)
- [ ] The panel shows a compact summary of the active route preferences
- [ ] Tapping "Discard" removes the route polyline and markers from the map and returns to the Route Planning Screen
- [ ] From and To field values are preserved after discard
- [ ] Tapping "Start navigation" either transitions to navigation or shows a "Coming soon" indication
- [ ] The bottom sheet cannot be dismissed by swipe
- [ ] The panel is correctly positioned and usable on a physical Android device
