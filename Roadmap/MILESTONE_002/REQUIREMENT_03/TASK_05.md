# Task 05 - Long-Press Pin Drop & Reverse Geocode Flow

**Status**: open

## Description

Register a long-press listener on the MapLibre map in `HomeScreen.kt`. On long-press,
immediately drop a pin at the pressed coordinate, trigger a reverse geocode call, and
display the resolved address as the pin label before showing the action sheet. If
geocoding fails, the pin label falls back to the formatted coordinate. Tapping elsewhere
on the map dismisses the pin and clears the action sheet.

### Pre-conditions

- TASK_02 is complete: `ReverseGeocodeRepository` (or `LocationSearchRepository`
  `reverseGeocode` method) is available.
- TASK_03 is complete: `SelectedLocation` and `RoutePlanningViewModel` are available.
- TASK_04 is complete: `LocationActionSheet` composable and `selectedLocation` state
  are in `HomeScreen`.

### Instructions

**Extend `LocationSearchViewModel` (or create `LongPressViewModel`)**

Add a dedicated method (prefer extending `LocationSearchViewModel` if it already
holds the repository, otherwise create a minimal `LongPressViewModel`):

```kotlin
private val _droppedPinState = MutableStateFlow<DroppedPinState>(DroppedPinState.Hidden)
val droppedPinState: StateFlow<DroppedPinState> = _droppedPinState.asStateFlow()

sealed class DroppedPinState {
    object Hidden : DroppedPinState()
    data class Loading(val lat: Double, val lon: Double) : DroppedPinState()
    data class Resolved(val location: SelectedLocation)  : DroppedPinState()
}

fun onLongPress(lat: Double, lon: Double) {
    _droppedPinState.value = DroppedPinState.Loading(lat, lon)
    viewModelScope.launch {
        val result = reverseGeocodeRepository.reverseGeocode(lat, lon)
        _droppedPinState.value = result.fold(
            onSuccess = { results ->
                val label = results.firstOrNull()?.label
                    ?: "%.5f, %.5f".format(lat, lon)
                DroppedPinState.Resolved(SelectedLocation(lat, lon, label))
            },
            onFailure = {
                DroppedPinState.Resolved(
                    SelectedLocation(lat, lon, "%.5f, %.5f".format(lat, lon))
                )
            }
        )
    }
}

fun dismissDroppedPin() {
    _droppedPinState.value = DroppedPinState.Hidden
}
```

**File: `HomeScreen.kt`**

- Wire `droppedPinState` from the ViewModel:

```kotlin
val droppedPinState by viewModel.droppedPinState.collectAsState()
```

- Register the long-press listener inside the `AndroidView` / `MapLibreMap` setup
  block:

```kotlin
map.addOnMapLongClickListener { latLng ->
    viewModel.onLongPress(latLng.latitude, latLng.longitude)
    true
}
```

- Register a regular map click listener to dismiss the pin when the user taps
  elsewhere (i.e., not on a symbol):

```kotlin
map.addOnMapClickListener { _ ->
    viewModel.dismissDroppedPin()
    selectedLocation = null
    false  // do not consume — allow symbol click listeners to fire first
}
```

- Observe `droppedPinState` with a `LaunchedEffect` to manage the `droppedPinSymbol`
  on the `symbolManager`:

```kotlin
LaunchedEffect(droppedPinState) {
    when (val state = droppedPinState) {
        is DroppedPinState.Hidden   -> removeDroppedPinSymbol()
        is DroppedPinState.Loading  -> addOrUpdateDroppedPinSymbol(
                                           state.lat, state.lon, label = "..."
                                       )
        is DroppedPinState.Resolved -> {
            addOrUpdateDroppedPinSymbol(state.location.lat, state.location.lon,
                                        label = state.location.label)
            selectedLocation = state.location   // triggers action sheet
        }
    }
}
```

- Implement `addOrUpdateDroppedPinSymbol` and `removeDroppedPinSymbol` as private
  helpers that add/update/remove a `Symbol` on the existing `symbolManager`.
- Use a distinct icon or style for the dropped pin to differentiate it from search
  result markers (a standard pin icon from the map style is acceptable).

### Test Scenarios

1. **Long-press → Loading state** — call `onLongPress(lat, lon)`; expect
   `droppedPinState` to immediately emit `Loading(lat, lon)`.
2. **Successful reverse geocode** — mock repository to return one result; expect
   `droppedPinState` to transition to `Resolved` with the result label.
3. **Failed reverse geocode** — mock repository to return `Result.failure`; expect
   `droppedPinState` to transition to `Resolved` with `"lat, lon"` formatted to
   5 decimal places.
4. **Empty results** — mock repository to return `Result.success(emptyList())`; expect
   fallback to coordinate label.
5. **`dismissDroppedPin`** — after `Resolved`, call `dismissDroppedPin()`; expect
   `droppedPinState` to emit `Hidden`.

## Acceptance Criteria

- [ ] Long-pressing the map immediately sets `droppedPinState` to `Loading` and places
      a pin on the map with a placeholder label.
- [ ] A reverse geocode request is sent for the long-pressed coordinate.
- [ ] On success, the pin label updates to the resolved address.
- [ ] On failure or empty results, the pin label shows `"lat, lon"` formatted to 5
      decimal places.
- [ ] Once `droppedPinState` is `Resolved`, `selectedLocation` is set and the
      `LocationActionSheet` is shown.
- [ ] Tapping elsewhere on the map calls `dismissDroppedPin`, removes the pin, and
      clears `selectedLocation`.
- [ ] Unit tests cover Loading → Resolved (success), Loading → Resolved (failure),
      Loading → Resolved (empty), and dismiss.
- [ ] The app compiles without errors.
