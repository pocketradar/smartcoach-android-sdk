# Changelog

All notable changes to the SmartCoach Android SDK. The SDK is in beta; APIs may change between releases.

## 0.2.0-beta.1

The first Android release. The API matches the iOS SDK 0.2.0-beta.1 name for name; the
differences below are the ones Android requires.

### Added

- `SmartCoach.configure`, `startScanning`, `stopScanning`, `connect`, `disconnect`,
  `startMeasuring`, `stopMeasuring`, `sessionStateStream`, `currentSessionState`,
  `getVersion`, as on iOS.
- Radar settings: `setMeasurementUnit`, `setSpeedRange`, `setSensitivity` with the
  validated value types `RadarSpeedRange` and `RadarSensitivity`. Calls wait for the
  radar's acknowledgement and throw `commandFailed` (4009) if it doesn't come.
- SmartCoach 1 and SmartCoach 2: discovery, connection, measuring, units and speed range.
  Sensitivity is SmartCoach 1 only. `startScanning(connectToLastPairedDevice = true)`
  connects to the radar this app last paired with as soon as it is found.
- The session follows the radar: measuring started or stopped from the radar's own
  trigger moves the session between `connected` and `measuring`.
- A connection that doesn't finish authenticating within 10 s fails with `failedToConnect`.
- Entitlements, as on iOS: the API key (manifest `<meta-data>`) decides which radar models
  the app may scan for, connect to and measure with. Calls it doesn't allow throw
  `featureNotAvailable` (3001); entitlements that can't be fetched or verified throw
  `invalidEntitlements` (3002). A key missing or not a UUID makes `configure` throw
  `missingApiKey` (1003). `SDKFeature` and `FeatureType` describe the features.
- Scanned radars report their model (`deviceType`).

### Android-specific

- Kotlin coroutines in place of Swift concurrency: `suspend` functions, `Flow` for
  readings, and `StateFlow` for the session state.
- `configure` takes a `Context`.
- The SDK runs a `connectedDevice` foreground service while a session is active,
  configured with `SmartCoachDeviceConfigurationOptions.notification`.
- `SmartCoachErrorCode.permissionDenied` (4010) when a Bluetooth runtime permission is
  missing.
- `SmartCoachRadar.name`: the name the radar advertises, matching the number it shows
  while pairing.
- `SmartCoach.measurementStream()`: readings in whatever mode the radar is in, without
  sending it a command.
- `SmartCoachDeviceConfigurationOptions.logger` routes SDK logs to your own logger.

- Entitlements fetched online stay usable offline for about 21 days, including after the
  app restarts; iOS currently needs network access again after a restart.
- If the entitlements service can't be reached (no response, a server error or rate
  limiting) while the phone is online, the SDK falls back to the last entitlements fetched;
  iOS fails with `invalidEntitlements`.

### Known limitations

- SmartCoach 2 readings report direction `unknown`, velocity type `generic`, and no tilt.

### Requirements

Android 10 (API 29)+, `compileSdk` 36+, Kotlin 2.1+, Android Gradle Plugin 8.0+.
