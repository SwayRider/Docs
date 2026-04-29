# REQUIREMENT_001 — Tasks Index

## Tasks

| ID | Name | Status | Description |
|----|------|--------|-------------|
| TASK_001 | Backend: RouteType Enum | Planned | Add Fastest/Shortest/Scenic enum to proto with preset preference mapping |
| TASK_002 | Mobile: API Client & Data Layer | Planned | RouterService Retrofit interface, DTOs, domain models, RoutingRepository |
| TASK_003 | Mobile: Route Planning ViewModel | Planned | RoutePlanningViewModel with state for origin/dest, preferences, and result |
| TASK_004 | Mobile: Route Planning UI | Planned | Origin/dest selection, route type chips, avoid toggles, route summary |
| TASK_005 | Mobile: Route Display on Map | Planned | Draw route polyline and markers on MapLibre map in HomeScreen |

## Task Dependencies

```
TASK_001 (RouteType proto) ──► TASK_002 (API client) ──► TASK_003 (ViewModel) ──► TASK_004 (UI) ──► TASK_005 (Map display)
```

## Notes

- Multi-region routing and border crossing: already implemented in RouterService
- Preference floats (highway/toll/ferry): already implemented in RouterService
- Motorcycle type (Sport/Touring/Cruiser): deferred to a later requirement
- Route saving/favorites: deferred to a later requirement
- Route recalculation on deviation: deferred to the navigation requirement

## Acceptance Criteria Summary

- [ ] RouteType enum in proto (Fastest, Shortest, Scenic)
- [ ] Scenic preset maps to low highway + high trail preferences
- [ ] RouterService API client wired in Android app
- [ ] Route planning screen with origin/dest, type selector, avoid toggles
- [ ] Route summary (distance, time) shown after calculation
- [ ] Route polyline drawn on map
