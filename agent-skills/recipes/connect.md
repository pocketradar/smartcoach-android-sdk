# Recipe: Connect & Disconnect

Add connecting to a discovered device (and disconnecting) to a ViewModel.

**Read `conventions.md` first** — including "contract vs. implementation" and Step 1
(detect the architecture and audit existing SmartCoach usage). Usually applied after
`scan.md` (you connect to a device the user picked from the scan list).

This recipe's **"Rules specific to connecting"** are the architecture-neutral contract.
The **"Adds"** Kotlin is the ViewModel reference — for another architecture, apply the
rules idiomatically per conventions.md's "Mapping the contract" instead of pasting it.

---

## Prerequisites

- The app depends on `com.pocketradar:smartcoach`, calls `SmartCoach.configure()` at
  launch, and has the Bluetooth runtime permissions.
- A device to connect to — typically from the `scanning` list (`scan.md`), or the last
  paired device via `startScanning(autoConnect = true)`.

## Adds (ViewModel reference implementation)

**No new state is required** — the connected device is carried in `sessionState`
(the `radar` of `connecting`/`connected`/`measuring`/`reconnecting`). Read it where you
need it, e.g. in the UI:
`(sessionState as? SmartCoachSessionState.connected)?.radar`.

**Branches in the collector's `when`** — connection states are handled by the canonical
`else` branch already (clear transient lists). Add explicit branches only if the UI must
react. Common addition:

```kotlin
is SmartCoachSessionState.connected -> _availableDevices.value = emptyList()   // stop showing the scan list
```

(`connecting` and `reconnecting` typically fall through to `else`.)

**Action functions** (using `launchCall` from conventions.md)

```kotlin
fun connectToDevice(device: SmartCoachRadar) {
    if (sessionState.value is SmartCoachSessionState.connected) return
    launchCall("Connection failed") { SmartCoach.connect(device) }
}

fun disconnect() {
    viewModelScope.launch { SmartCoach.disconnect() }
}
```

---

## Rules specific to connecting

- **`connect()` returning does NOT mean "ready."** The Bluetooth link is starting and the
  encryption handshake follows; the session only becomes `connected` afterward, via the
  stream. Anything that needs a ready device (e.g. `startMeasuring`, settings) must wait
  for the `connected` state, not for `connect()` to return. A handshake that fails or
  doesn't finish within 10 seconds ends in `disconnected` with `failedToConnect`.
- **`disconnect()` never throws.** No `try`/`catch` needed. If an error occurred it
  arrives as `disconnected(error)` on the stream.
- **Connecting while already connected/connecting throws `invalidSessionState`.** The
  guard above avoids the common case.
- **Connecting needs the `connect` entitlement for the radar's model**, otherwise
  `connect()` throws `featureNotAvailable`. A radar whose model turns out, once connected,
  not to be covered is dropped: the session ends in `disconnected` with
  `noDeviceConnected`. The same check applies to auto-connect and auto-reconnect.
- **Reconnection is automatic only for *unexpected* disconnects**, and only when the SDK
  was configured with `autoReconnect = true`. The SDK drives it and surfaces
  `reconnecting(radar)` on the stream; show it in the UI if you want, but do not call
  `connect` yourself to "help." A user-initiated `disconnect()` never auto-reconnects.
- **The connection outlives the screen.** While connected, the SDK runs a foreground
  service with an ongoing notification, so the connection survives the app going to the
  background and the ViewModel being cleared. Call `disconnect()` when the user is done,
  not in `onCleared()` — unless the app wants the radar dropped whenever this screen goes
  away.

---

## Composability

- Add the two action functions.
- Optionally add the `connected` branch (to hide the scan list); otherwise `else`
  handles it.
- If `scan.md` is present, `connectToDevice()` pairs with the scan list's "Connect"
  buttons.
- If `measure.md` or `settings.md` is present, the `connected` state is their gate.

---

## Minimal illustrative UI fragment (adapt — not required UI)

```kotlin
// A "Connect" button per scanned device:
Button(onClick = { viewModel.connectToDevice(device) }) { Text(stringResource(R.string.connect)) }

// A single action button reacting to state:
when (sessionState) {
    is SmartCoachSessionState.disconnected ->
        Button(onClick = { viewModel.startScanning() }) { Text(stringResource(R.string.scan)) }
    is SmartCoachSessionState.scanning ->
        Button(onClick = viewModel::stopScanning) { Text(stringResource(R.string.stop_scanning)) }
    is SmartCoachSessionState.connecting,
    is SmartCoachSessionState.connected,
    is SmartCoachSessionState.measuring,
    is SmartCoachSessionState.reconnecting ->
        Button(onClick = viewModel::disconnect) { Text(stringResource(R.string.disconnect)) }
}
```
