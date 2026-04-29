# Task 03 - SelectedLocation Domain Class & RoutePlanningViewModel

**Status**: open

## Description

Introduce the `SelectedLocation` domain data class and `RoutePlanningViewModel` that
act as the shared state bridge between `HomeScreen` (writes) and `RoutePlanningScreen`
(reads). Wire both into the DI container in `SwayRiderApp.kt` and thread the ViewModel
through `SwayRiderNavHost`.

### Instructions

**Domain data class**

Create `SelectedLocation` in the domain layer (alongside other domain models):

```kotlin
data class SelectedLocation(
    val lat: Double,
    val lon: Double,
    val label: String,
)
```

**File: `RoutePlanningViewModel.kt`** (new file)

```kotlin
class RoutePlanningViewModel : ViewModel() {

    private val _pendingOrigin      = MutableStateFlow<SelectedLocation?>(null)
    private val _pendingDestination = MutableStateFlow<SelectedLocation?>(null)

    val pendingOrigin:      StateFlow<SelectedLocation?> = _pendingOrigin.asStateFlow()
    val pendingDestination: StateFlow<SelectedLocation?> = _pendingDestination.asStateFlow()

    /** Sets the destination; origin = null means "use current GPS location". */
    fun setNavigateTo(destination: SelectedLocation) {
        _pendingOrigin.value      = null
        _pendingDestination.value = destination
    }

    fun setOrigin(location: SelectedLocation) {
        _pendingOrigin.value = location
    }

    fun setDestination(location: SelectedLocation) {
        _pendingDestination.value = location
    }

    fun clear() {
        _pendingOrigin.value      = null
        _pendingDestination.value = null
    }
}
```

**File: `SwayRiderApp.kt`**

- Inside the `diContainer` `remember` block, instantiate `RoutePlanningViewModel`
  following the same pattern as `locationSearchViewModel`:

```kotlin
val routePlanningViewModel = remember { RoutePlanningViewModel() }
```

- Pass `routePlanningViewModel` as a parameter to `SwayRiderNavHost`.

**File: `SwayRiderNavHost.kt`** (or wherever the nav host is defined)

- Accept `routePlanningViewModel: RoutePlanningViewModel` as a parameter.
- Forward it to both `HomeScreen` (write) and `RoutePlanningScreen` (read) via their
  composable call-sites within the nav graph.
- `HomeScreen` will use it to write `pendingOrigin` / `pendingDestination`.
- `RoutePlanningScreen` will use it to read pre-filled values (reading is not
  implemented in this task — just threading the parameter is sufficient).

### Test Scenarios

1. **`setNavigateTo`** — call `setNavigateTo(loc)`; expect `pendingDestination == loc`
   and `pendingOrigin == null`.
2. **`setOrigin`** — call `setOrigin(loc)`; expect `pendingOrigin == loc`,
   `pendingDestination` unchanged.
3. **`setDestination`** — call `setDestination(loc)`; expect `pendingDestination == loc`,
   `pendingOrigin` unchanged.
4. **`clear`** — after setting both, call `clear()`; expect both flows emit `null`.
5. **`StateFlow` emissions** — verify each mutating method emits the new value to
   collectors.

## Acceptance Criteria

- [ ] `SelectedLocation(lat, lon, label)` domain data class exists.
- [ ] `RoutePlanningViewModel` exposes `pendingOrigin` and `pendingDestination` as
      `StateFlow<SelectedLocation?>`.
- [ ] `setNavigateTo`, `setOrigin`, `setDestination`, and `clear` behave as specified.
- [ ] `RoutePlanningViewModel` is instantiated in `SwayRiderApp.kt` `diContainer`.
- [ ] `SwayRiderNavHost` accepts and forwards `routePlanningViewModel` to both
      `HomeScreen` and `RoutePlanningScreen`.
- [ ] Unit tests cover all four ViewModel methods.
- [ ] The app compiles without errors.
