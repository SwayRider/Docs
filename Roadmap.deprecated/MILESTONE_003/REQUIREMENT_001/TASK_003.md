# TASK_003 — Mobile: Route Planning ViewModel

**Status**: Planned

## Overview

Implement `RoutePlanningViewModel` to manage the full state of the route planning flow: origin/destination, route preferences, calculation state, and result.

## Repository

- **Repo**: swayrider
- **Subfolder**: `mobile/android/`
- **Tech**: Kotlin, ViewModel, StateFlow, Jetpack Compose

## Background

The pattern to follow is `LocationSearchViewModel` in `ui/viewmodel/` (or `viewmodel/`): StateFlow-based reactive state, suspend functions called from `viewModelScope`, sealed UI state class.

## Technical Specification

### State (`mobile/android/app/src/main/java/.../viewmodel/RoutePlanningViewModel.kt`)

```kotlin
data class RoutePlanningState(
    val origin: RouteLocation? = null,
    val originLabel: String = "",
    val destination: RouteLocation? = null,
    val destinationLabel: String = "",
    val preferences: RoutePreferences = RoutePreferences(),
    val calculationState: RouteCalculationState = RouteCalculationState.Idle
)

sealed class RouteCalculationState {
    object Idle : RouteCalculationState()
    object Loading : RouteCalculationState()
    data class Success(val result: RouteResult) : RouteCalculationState()
    data class Error(val message: String) : RouteCalculationState()
}
```

### ViewModel

```kotlin
class RoutePlanningViewModel(
    private val routingRepository: RoutingRepository,
    private val locationSearchRepository: LocationSearchRepository  // reuse for label resolution
) : ViewModel() {

    private val _state = MutableStateFlow(RoutePlanningState())
    val state: StateFlow<RoutePlanningState> = _state.asStateFlow()

    fun setOrigin(location: RouteLocation, label: String)
    fun setDestination(location: RouteLocation, label: String)
    fun setOriginFromGps(lat: Double, lng: Double)   // uses "My Location" label
    fun swapOriginDestination()
    fun updatePreferences(preferences: RoutePreferences)
    fun calculateRoute()    // calls repository, updates calculationState
    fun clearRoute()        // resets calculationState to Idle
}
```

### DI Wiring (`SwayRiderApp.kt`)

```kotlin
val routePlanningViewModel = RoutePlanningViewModel(
    routingRepository = routingRepository,
    locationSearchRepository = locationSearchRepository
)
```

Pass to `RoutePlanningScreen` the same way other ViewModels are currently passed.

## Dependencies

- TASK_002 (RoutingRepository must exist)

## Acceptance Criteria

- [ ] `RoutePlanningState` data class with all required fields
- [ ] `RouteCalculationState` sealed class (Idle, Loading, Success, Error)
- [ ] `RoutePlanningViewModel` implements all listed functions
- [ ] `calculateRoute()` calls repository and updates state correctly
- [ ] Error state populated on network/API failure
- [ ] ViewModel wired into DI in `SwayRiderApp.kt`
- [ ] Code compiles

## Testing Notes

Unit test `calculateRoute()` with a mock repository returning success and error cases. Verify StateFlow emits the correct state transitions.
