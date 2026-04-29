# Requirement 04 - Route Planning Screen

**Status**: Open

**Scope**:
  - `mobile/android/` — route planning screen, search overlay, location provider

## Goal

Provide a dedicated Route Planning Screen where the user can set origin and destination, use their current location as origin, swap origin and destination, and select motorcycle-specific route preferences before triggering route calculation.

## Description

The Route Planning Screen is the primary UI for configuring a route. It is opened from:
- The map action sheet ("Navigate to", "Set as destination", "Set as start point")
- A direct "Plan route" entry point on the home screen (if applicable)

### From/To Input Bar

A persistent input bar at the top of the screen contains two fields: **From** and **To**.

- Each field shows the resolved address of the selected location, or a placeholder ("Choose start point" / "Choose destination") if not yet set.
- Tapping either field opens a **search overlay** (full-screen search input with results list, reusing the existing search component).
- A **swap button** between the two fields swaps origin and destination.

### My Current Location

"My current location" appears as the first option in the From field's search overlay. Selecting it uses the device GPS location as the origin. The field label shows "My location" with a location icon.

### Route Preferences

Below the From/To bar, a preferences row allows the user to toggle:
- Scenic roads (prefer winding roads)
- Avoid highways
- Avoid tolls
- Unpaved roads: prefer / neutral / avoid (segmented control or dropdown)

Preferences are persisted across sessions (DataStore or SharedPreferences).

### Calculate Button

A "Calculate route" button is enabled only when both From and To are set. Tapping it triggers route calculation (covered in REQUIREMENT_05 and REQUIREMENT_06). During calculation a loading state is shown on the button.

## Tasks

_Tasks to be defined._

## Dependencies

- MILESTONE_001 (Foundation Complete) — search component and location services must be in place
- REQUIREMENT_03 (Map Interaction & Waypoint Selection) — action sheet pre-fills the planning screen fields

## Acceptance Criteria

- [ ] A Route Planning Screen is accessible from the map action sheet and opens with the pre-filled field(s) from the action sheet selection
- [ ] Tapping the From or To field opens a search overlay
- [ ] Selecting a search result closes the overlay and populates the field with the resolved address
- [ ] "My current location" appears as the first option in the From search overlay and uses device GPS
- [ ] The swap button exchanges the From and To values
- [ ] All four motorcycle preference controls are present and functional
- [ ] Route preferences are persisted and restored between app sessions
- [ ] The "Calculate route" button is disabled when either From or To is empty
- [ ] The "Calculate route" button shows a loading state while a calculation is in progress
- [ ] The screen works correctly on a physical Android device
