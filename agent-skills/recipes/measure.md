# Recipe: Stream Measurements

Add live speed measurement streaming to a ViewModel.

**Read `conventions.md` first** — including "contract vs. implementation" and Step 1
(detect the architecture and audit existing SmartCoach usage). Applied after
`connect.md` — you can only measure a connected device.

This recipe's **"Rules specific to measuring"** are the architecture-neutral contract.
The **"Adds"** Kotlin is the ViewModel reference — for another architecture, apply the
rules idiomatically per conventions.md's "Mapping the contract" instead of pasting it.
The cross-capability cleanup (cancel the readings job in non-measuring states) is part
of the contract, however the `when`/reducer is expressed.

---

## Prerequisites

- The app depends on `com.pocketradar:smartcoach` and calls `SmartCoach.configure()` at
  launch.
- A `connected` or `measuring` device (see `connect.md`). Measuring from any other
  state (`connecting`, `reconnecting`, `scanning`, `disconnected`) throws
  `invalidSessionState`.

## Adds (ViewModel reference implementation)

**State**

```kotlin
private val _speeds = MutableStateFlow<List<MeasurementData>>(emptyList())
/** Latest reading first: .measurement (value + unit), .direction, .velocityType, .tilt. */
val speeds: StateFlow<List<MeasurementData>> = _speeds.asStateFlow()

private var speedsJob: Job? = null
```

Unlike the session collector, the readings collection is a **stored** `Job` (cancelled on
state transitions), because it starts and stops independently of the ViewModel's
lifetime.

**Branches in the collector's `when`** — measuring introduces cross-capability cleanup.
Add a `measuring` branch that does NOT cancel the readings job, and ensure the
non-measuring branches DO cancel it:

```kotlin
is SmartCoachSessionState.measuring -> {
    _availableDevices.value = emptyList()       // keep the readings job running
    if (speedsJob?.isActive != true) {          // the radar started on its own — subscribe to its readings
        collectReadings(reportFailure = false)
    }
}
```

Then add `cancelReadings()` to the sibling branches that end measurement — `scanning`,
`disconnected`, and `else` (and `connected`, if `connect.md` added one):

```kotlin
is SmartCoachSessionState.scanning -> {
    _availableDevices.value = state.radars
    cancelReadings()
}
is SmartCoachSessionState.disconnected -> {
    _availableDevices.value = emptyList()
    cancelReadings()
    state.error?.let { _errorMessage.value = it.message }
}
else -> {
    _availableDevices.value = emptyList()
    cancelReadings()
}
```

**Action functions & helpers**

```kotlin
fun startMeasuring() {
    // Valid from connected (starts the radar) or measuring (radar already running — the
    // flow just delivers readings, no command is sent). Anything else throws
    // invalidSessionState; the check keeps the UI honest.
    val state = sessionState.value
    if (state !is SmartCoachSessionState.connected && state !is SmartCoachSessionState.measuring) return
    if (speedsJob?.isActive != true) collectReadings(reportFailure = true)
}

fun stopMeasuring() {
    cancelReadings()
    if (sessionState.value !is SmartCoachSessionState.measuring) return
    launchCall("Failed to stop measuring") { SmartCoach.stopMeasuring() }
}

private fun collectReadings(reportFailure: Boolean) {
    _speeds.value = emptyList()
    speedsJob = viewModelScope.launch {
        try {
            // The flow completes when measuring stops (by the app or by the radar), the
            // device disconnects, or the connection is lost — collect returns on its own.
            SmartCoach.startMeasuring().collect { reading ->
                _speeds.update { listOf(reading) + it }
            }
        } catch (e: SmartCoachError) {
            if (reportFailure) _errorMessage.value = "Failed to start measuring: ${e.message}"
        }
    }
}

private fun cancelReadings() {
    speedsJob?.cancel()
    speedsJob = null
    _speeds.value = emptyList()
}
```

(Imports: `kotlinx.coroutines.Job`, `kotlinx.coroutines.flow.update`,
`com.pocketradar.smartcoach.MeasurementData`.) Cap the list (e.g. `.take(50)`) if the UI
only shows recent readings.

---

## Rules specific to measuring

- **`startMeasuring()` requires `connected` or `measuring`.** Not `connecting`, not
  "right after `connect()` returned." The SDK throws `invalidSessionState` otherwise.
  From `measuring` it sends nothing: you just get the readings.
- **Measuring needs the `measurement` entitlement for the radar's model**, otherwise
  `startMeasuring()` throws `featureNotAvailable`. Report it (the reference code does when
  the user asked to measure); hide the measuring controls if the app knows its key lacks it.
- **The radar can start measuring by itself** (its trigger). The session becomes
  `measuring` without an app call — that is why the `measuring` branch subscribes when no
  readings job is running. Never treat an unexpected `measuring` as an error.
- **Collect the returned flow right away.** Readings that arrive before collection starts
  are not kept, so don't store the flow to collect later.
- **The readings flow completes by itself** on `stopMeasuring()`, `disconnect()`, the
  radar stopping on its own, or a dropped connection. The stored `speedsJob` +
  `cancelReadings()` handle the cases where *state changes out from under* an active
  measurement.
- **Only collect in a scope that ends with its owner** (`viewModelScope` here). A
  `GlobalScope` collection would keep running after the screen is gone.
- **Readings carry their own unit.** `reading.measurement.value` is in
  `reading.measurement.unit` (the radar's current unit); use
  `reading.measurement.converted(RadarMeasurementUnit.kph)` to show a fixed unit.

---

## Composability

- Add `speeds`, `speedsJob`, the two action functions, the two helpers, and the
  `measuring` branch.
- **Critically**, inject `cancelReadings()` into the existing non-measuring branches
  (`scanning`, `disconnected`, `else`, and any explicit `connected`/`connecting`/
  `reconnecting` branch). This is the cross-capability cleanup called out in
  `conventions.md` — measuring is the one recipe that edits sibling branches.
- Requires `connect.md`'s `connected` state to be reachable; pairs with a
  "Start/Stop measuring" button shown when connected/measuring.

---

## Minimal illustrative UI fragment (adapt — not required UI)

```kotlin
val speeds by viewModel.speeds.collectAsStateWithLifecycle()

when (sessionState) {
    is SmartCoachSessionState.connected ->
        Button(onClick = viewModel::startMeasuring) { Text(stringResource(R.string.start_measuring)) }
    is SmartCoachSessionState.measuring ->
        Button(onClick = viewModel::stopMeasuring) { Text(stringResource(R.string.stop_measuring)) }
    else -> Unit
}
LazyColumn {
    items(speeds.take(10)) { reading ->
        Row {
            Text("%.1f %s".format(reading.measurement.value, reading.measurement.unit),
                fontFamily = FontFamily.Monospace)
            Spacer(Modifier.weight(1f))
            Text("${reading.direction} · ${reading.velocityType}",
                style = MaterialTheme.typography.bodySmall)
        }
    }
}
```
