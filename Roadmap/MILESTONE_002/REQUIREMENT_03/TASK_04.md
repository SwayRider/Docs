# Task 04 - Search Result Marker Tap → Action Sheet

**Status**: open

## Description

When the user taps an existing search result marker on the map, a `ModalBottomSheet`
action sheet appears with three options: "Navigate to", "Set as destination", and
"Set as start point". Each option writes to `RoutePlanningViewModel` and navigates
to the Route Planning Screen. Swiping down or tapping the scrim dismisses the sheet.

### Pre-conditions

- TASK_03 is complete: `SelectedLocation` and `RoutePlanningViewModel` are available,
  and `HomeScreen` already receives `routePlanningViewModel` as a parameter.

### Instructions

**File: `HomeScreen.kt`**

- Add local state to drive action sheet visibility:

```kotlin
var selectedLocation by remember { mutableStateOf<SelectedLocation?>(null) }
```

- After the existing `symbolManager` setup, register a click listener on the
  `searchSymbol` layer:

```kotlin
symbolManager.addClickListener { symbol ->
    val loc = SelectedLocation(
        lat   = symbol.latLng.latitude,
        lon   = symbol.latLng.longitude,
        label = symbol.textField ?: "",
    )
    selectedLocation = loc
    true  // consume the event
}
```

- At the bottom of the `HomeScreen` composable (outside the `AndroidView`), add:

```kotlin
if (selectedLocation != null) {
    LocationActionSheet(
        location             = selectedLocation!!,
        onNavigateTo         = {
            routePlanningViewModel.setNavigateTo(selectedLocation!!)
            selectedLocation = null
            navController.navigate(Screen.RoutePlanning.route)
        },
        onSetAsDestination   = {
            routePlanningViewModel.setDestination(selectedLocation!!)
            selectedLocation = null
            navController.navigate(Screen.RoutePlanning.route)
        },
        onSetAsStartPoint    = {
            routePlanningViewModel.setOrigin(selectedLocation!!)
            selectedLocation = null
            navController.navigate(Screen.RoutePlanning.route)
        },
        onDismiss            = { selectedLocation = null },
    )
}
```

**New composable: `LocationActionSheet`**

Create this composable in a new file (e.g., `LocationActionSheet.kt`) in the same
UI package as `HomeScreen`:

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun LocationActionSheet(
    location:           SelectedLocation,
    onNavigateTo:       () -> Unit,
    onSetAsDestination: () -> Unit,
    onSetAsStartPoint:  () -> Unit,
    onDismiss:          () -> Unit,
) {
    ModalBottomSheet(onDismissRequest = onDismiss) {
        // Header
        Text(
            text  = location.label,
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
        )
        HorizontalDivider()

        // Action rows
        ActionRow(label = "Navigate to",       onClick = onNavigateTo)
        ActionRow(label = "Set as destination", onClick = onSetAsDestination)
        ActionRow(label = "Set as start point", onClick = onSetAsStartPoint)

        Spacer(modifier = Modifier.height(16.dp))
    }
}
```

- Implement a private `ActionRow(label, onClick)` helper composable with a
  `ListItem` (or `Row` + `Text`) that fills the width and handles click.
- The `ModalBottomSheet` is already imported in the project (Material3).

### Test Scenarios

1. **`LocationActionSheet` renders correctly** — snapshot or Compose UI test verifying
   the label text and three action row labels appear when `selectedLocation` is non-null.
2. **"Navigate to" tap** — simulate click; expect `onNavigateTo` lambda invoked once,
   and `onDismiss` NOT called separately.
3. **"Set as destination" tap** — simulate click; expect `onSetAsDestination` invoked.
4. **"Set as start point" tap** — simulate click; expect `onSetAsStartPoint` invoked.
5. **Scrim / swipe dismiss** — trigger `onDismissRequest`; expect `onDismiss` invoked
   and sheet disappears.

## Acceptance Criteria

- [ ] Tapping a search result marker sets `selectedLocation` state and triggers the
      `LocationActionSheet`.
- [ ] `LocationActionSheet` composable displays the location label and three labelled
      action rows.
- [ ] Each action row calls the correct `RoutePlanningViewModel` method and navigates
      to `Screen.RoutePlanning`.
- [ ] After any action or dismiss, `selectedLocation` is reset to `null`.
- [ ] Swiping down or tapping the scrim dismisses the sheet via `onDismissRequest`.
- [ ] Compose UI tests cover rendering and all three action taps.
- [ ] The app compiles without errors.
