# Recipe: Radar Settings (units, speed range, sensitivity)

Add controls that change settings on the connected radar.

**Read `conventions.md` first** — including "contract vs. implementation" and Step 1
(detect the architecture and audit existing SmartCoach usage). Applied after
`connect.md`: settings need a connected radar.

This recipe's **"Rules specific to settings"** are the architecture-neutral contract.
The **"Adds"** Kotlin is the ViewModel reference — for another architecture, apply the
rules idiomatically per conventions.md's "Mapping the contract" instead of pasting it.

---

## Prerequisites

- The app depends on `com.pocketradar:smartcoach` and calls `SmartCoach.configure()` at
  launch.
- The session is `connected` or `measuring`. Settings calls from any other state throw
  `invalidSessionState`.

## The three settings

| Call | Value type | Effect | Devices |
|---|---|---|---|
| `SmartCoach.setMeasurementUnit(unit)` | `RadarMeasurementUnit` (`mph` / `kph`) | The radar displays and reports in this unit; `MeasurementData.measurement` follows it. | SmartCoach 1 & 2 |
| `SmartCoach.setSpeedRange(range)` | `RadarSpeedRange` | The radar ignores readings outside the window. | SmartCoach 1 & 2 |
| `SmartCoach.setSensitivity(sensitivity)` | `RadarSensitivity` | Detection sensitivity, 1 (least) … 10 (default). | SmartCoach 1 only |

`RadarSpeedRange(lowMPH, highMPH)` and `RadarSensitivity(level)` **return `null`** for
values outside the hardware limits (`25 ≤ low ≤ high ≤ 130` mph; level `1…10`). A `null`
result means the user's input is out of range — report it in the UI; there is nothing to
send. `RadarSpeedRange.full`, `RadarSpeedRange.minimumMPH`/`maximumMPH`,
`RadarSensitivity.default` and `RadarSensitivity.minimumLevel`/`maximumLevel` give the
bounds for sliders and steppers.

## Adds (ViewModel reference implementation)

**State**

```kotlin
private val _settingsMessage = MutableStateFlow<String?>(null)
val settingsMessage: StateFlow<String?> = _settingsMessage.asStateFlow()
```

**Action functions**

```kotlin
fun setUnit(unit: RadarMeasurementUnit) =
    runSetting("Units") { SmartCoach.setMeasurementUnit(unit) }

fun setSpeedRange(lowMPH: Int, highMPH: Int) {
    val range = RadarSpeedRange(lowMPH, highMPH) ?: run {
        _errorMessage.value = "Speed range must satisfy 25 ≤ low ≤ high ≤ 130 mph"
        return
    }
    runSetting("Speed range") { SmartCoach.setSpeedRange(range) }
}

fun setSensitivity(level: Int) {
    val sensitivity = RadarSensitivity(level) ?: run {
        _errorMessage.value = "Sensitivity must be 1…10"
        return
    }
    runSetting("Sensitivity") { SmartCoach.setSensitivity(sensitivity) }
}

private fun runSetting(label: String, apply: suspend () -> Unit) {
    _settingsMessage.value = "$label: waiting for the radar…"
    viewModelScope.launch {
        _settingsMessage.value = try {
            apply()
            "$label: applied"
        } catch (e: SmartCoachError) {
            when (e.code) {
                SmartCoachErrorCode.commandFailed -> "$label: the radar did not accept the change"
                SmartCoachErrorCode.featureNotSupported -> "$label: not supported on this radar"
                else -> {
                    _errorMessage.value = e.message
                    "$label: failed"
                }
            }
        }
    }
}
```

(Imports: `RadarMeasurementUnit`, `RadarSpeedRange`, `RadarSensitivity`,
`SmartCoachErrorCode` from `com.pocketradar.smartcoach`.) Put the user-facing messages in
`strings.xml` following the app's convention; they are inline here for readability.

**Reading the current values.** The radar's unit is on the device in the session state
(`radar.measurementUnit`) and updates once the radar confirms — read it from
`sessionState`, don't cache it. The SDK does not expose the current range or
sensitivity; the app owns those values.

---

## Rules specific to settings

- **Each call waits for the radar's acknowledgement** and returns only when the radar
  has confirmed (or throws `commandFailed` after about 3 seconds). Show a pending state.
  Exception: SmartCoach 2 firmware doesn't confirm speed-range changes, so on SmartCoach 2
  `setSpeedRange` returns once the command is sent.
- **Radars reset to defaults on every connection** (full range, default sensitivity, and
  the unit the radar was last left in). If the app has a preferred range/sensitivity,
  re-apply it in the `connected` branch of the collector — including after an
  auto-reconnect, which also lands in `connected`. Guard it so it runs once per
  connection, not on every `connected` emission (the radar snapshot is re-emitted when
  its battery or unit changes).
- **Validate with the value types' factories**, never by hand; the SDK will not send an
  invalid value.
- **`setSensitivity` throws `featureNotSupported` on SmartCoach 2.** Hide or disable the
  control when `radar.deviceType == DeviceType.smartCoach2`.
- **One setting at a time.** Calls are serialized inside the SDK (a second call waits for
  the first); awaiting them in sequence is fine, but don't fire several concurrently from
  the UI and expect ordering.
- **If the radar disconnects while a setting is pending**, the call throws
  `noDeviceConnected`.

---

## Composability

- Add `settingsMessage`, the three action functions, and `runSetting`.
- Re-apply persisted settings in the existing `connected` branch (do not add a second
  collector).
- Pairs with `connect.md` (needs `connected`) and is independent of `measure.md`.

---

## Minimal illustrative UI fragment (adapt — not required UI)

```kotlin
val radar = (sessionState as? SmartCoachSessionState.connected)?.radar
if (radar != null) {
    Row {
        Button(onClick = { viewModel.setUnit(RadarMeasurementUnit.mph) },
            enabled = radar.measurementUnit != RadarMeasurementUnit.mph) { Text("mph") }
        Button(onClick = { viewModel.setUnit(RadarMeasurementUnit.kph) },
            enabled = radar.measurementUnit != RadarMeasurementUnit.kph) { Text("kph") }
    }
    Button(onClick = { viewModel.setSpeedRange(40, 100) }) { Text("40–100 mph") }
    if (radar.deviceType == DeviceType.smartCoach1) {
        Button(onClick = { viewModel.setSensitivity(7) }) { Text("Sensitivity 7") }
    }
    settingsMessage?.let { Text(it, style = MaterialTheme.typography.bodySmall) }
}
```
