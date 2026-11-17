# Recipe: Device Scanning

Add SmartCoach device discovery to a ViewModel — a live list of nearby devices plus
start/stop scanning.

**Read `conventions.md` first** — including "contract vs. implementation" and Step 1
(detect the architecture and audit existing SmartCoach usage).

This recipe's **"Rules specific to scanning"** are the architecture-neutral contract.
The **"Adds"** Kotlin is the ViewModel reference implementation — for another
architecture, apply the rules idiomatically per conventions.md's "Mapping the contract"
instead of pasting it.

---

## Prerequisites

- The app depends on `com.pocketradar:smartcoach` and calls `SmartCoach.configure()` once
  at launch (otherwise scanning throws `notConfigured`).
- The Bluetooth runtime permissions are granted before scanning (otherwise it throws
  `permissionDenied`). See `configure.md` Step 6.

## Adds (ViewModel reference implementation)

**State**

```kotlin
private val _availableDevices = MutableStateFlow<List<SmartCoachRadar>>(emptyList())
val availableDevices: StateFlow<List<SmartCoachRadar>> = _availableDevices.asStateFlow()
```

**Branches in the `sessionState` collector's `when`**

```kotlin
is SmartCoachSessionState.scanning -> _availableDevices.value = state.radars
```

Also clear stale results when leaving the scanning context — add
`_availableDevices.value = emptyList()` to the `disconnected` branch and to `else` (and to
any other non-scanning branch a sibling recipe introduces).

**Action functions** (using `launchCall` from conventions.md)

```kotlin
fun startScanning(autoConnect: Boolean = false) {
    if (sessionState.value is SmartCoachSessionState.scanning) return
    launchCall("Failed to start scan") {
        SmartCoach.startScanning(connectToLastPairedDevice = autoConnect)
    }
}

fun stopScanning() = launchCall("Failed to stop scan") { SmartCoach.stopScanning() }
```

`autoConnect = true` asks the SDK to connect automatically to the radar this app last
paired with if it is found during the scan. `startScanning` also takes an optional
`timeout: Duration` (`kotlin.time`) after which scanning stops by itself.

---

## Rules specific to scanning

- **Devices arrive via `scanning(radars)`, never as a return value** from
  `startScanning()`. Each emission is the full current list; replace, don't append.
- **A device is identified by `id`**, not by object equality: the SDK emits a new
  `SmartCoachRadar` snapshot whenever its signal strength or other details change. Use
  `id` as the list key. `name` (e.g. `SC-123456`) matches the number the radar shows while
  pairing — the best label for users.
- **The list only shows radars in pairing mode** (or the paired radar, with
  `autoConnect`). A radar that is on but not pairing doesn't appear.
- **Scanning is invalid while connected/connecting** — `startScanning()` throws
  `invalidSessionState`. Surface it; don't retry blindly.
- **`stopScanning()` is safe** — a no-op if not scanning, and it never drops an
  established connection.
- **Missing runtime permissions throw `permissionDenied`, Bluetooth off throws
  `bluetoothNotAvailable`.** Route the first to the permission request, the second to a
  "turn on Bluetooth" message.
- **The list only shows radar models the API key covers.** A radar of another model never
  appears, even in pairing mode. `startScanning()` throws `featureNotAvailable` if the key
  covers no model, and `invalidEntitlements` if the entitlements can't be fetched (the
  first scan needs network access; afterwards they work offline for about 21 days).

---

## Composability

- Applied alone: it creates the canonical structure (from `conventions.md`) plus the
  above.
- Applied to a ViewModel that already collects the stream: add `availableDevices`, add
  the `scanning` branch, ensure non-scanning branches clear `availableDevices`, and add the
  two action functions. Do not add a second collector.

---

## Minimal illustrative UI (adapt to the app's design — not required UI)

```kotlin
@Composable
fun DeviceScanScreen(viewModel: RadarViewModel = viewModel()) {
    val sessionState by viewModel.sessionState.collectAsStateWithLifecycle()
    val devices by viewModel.availableDevices.collectAsStateWithLifecycle()

    Column {
        if (sessionState is SmartCoachSessionState.scanning) {
            Button(onClick = viewModel::stopScanning) { Text(stringResource(R.string.stop)) }
        } else {
            Button(onClick = { viewModel.startScanning() }) { Text(stringResource(R.string.scan)) }
        }
        LazyColumn {
            items(devices, key = { it.id }) { device ->
                Column {
                    Text(device.name, style = MaterialTheme.typography.titleMedium)
                    Text("RSSI ${device.rssi}", style = MaterialTheme.typography.bodySmall)
                }
            }
        }
    }
}
```
