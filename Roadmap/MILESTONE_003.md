# Milestone 003 - Turn-by-Turn Navigation

**Status**: Planned

## Goals

Deliver active turn-by-turn GPS navigation on the Android app. The user starts navigation from a planned route and receives real-time guidance: current maneuver instruction, distance to next turn, voice prompts, and automatic re-routing when off-track. The map follows the rider's position and heading.

## Key Deliverables

### GPS Position Tracking

The app continuously tracks the device GPS position during navigation. Location updates are fed into the navigation engine at a configurable rate. Background location tracking keeps navigation alive when the screen is off or another app is in the foreground.

### Navigation Engine (Android)

A navigation engine runs on the device and consumes the route geometry plus turn-by-turn instructions from RouterService. It computes:
- Current position on route (map matching)
- Distance and estimated time to next maneuver
- Distance and estimated time to destination
- Off-route detection

### Maneuver Instructions UI

The navigation screen displays the current and upcoming maneuver as a large, glanceable instruction panel (direction icon + street name + distance). A secondary panel shows the next maneuver. The map auto-rotates to heading and pans to follow the rider.

### Voice Guidance

Text-to-speech voice prompts announce upcoming maneuvers at configurable distances (e.g. 500m, 200m, at turn). Uses the Android TTS engine. Voice can be muted by the user.

### Off-Route Re-Routing

When the rider deviates from the route, the app detects this and automatically requests a new route from RouterService from the current position to the original destination. The new route is displayed and navigation resumes.

### Navigation End State

Navigation ends when the rider reaches the destination or manually stops. The app returns to the map/planning screen and shows a ride summary (total distance, ride time).

## Dependencies

- MILESTONE_002 (Route Planning) — route calculation, RouterService API, and route display must be complete

## Acceptance Criteria

- [ ] GPS position is tracked continuously during navigation
- [ ] Navigation persists in background (screen off, app backgrounded)
- [ ] Current maneuver instruction and distance are displayed clearly
- [ ] Map follows rider position and rotates to heading
- [ ] Voice guidance announces maneuvers via Android TTS
- [ ] Voice guidance can be muted
- [ ] Off-route detection triggers automatic re-routing
- [ ] Re-routed route is displayed and navigation resumes seamlessly
- [ ] Navigation ends correctly at destination with ride summary
- [ ] Navigation tested end-to-end on a physical Android device
