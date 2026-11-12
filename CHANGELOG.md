# Changelog

All notable changes to the SmartCoach Android SDK. The SDK is in beta; APIs may change between releases.

## Unreleased — first beta

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

### Known limitations

- SmartCoach 2 readings report direction `unknown`, velocity type `generic`, and no tilt.

### Requirements

Android 10 (API 29)+, `compileSdk` 36+, Kotlin 2.1+, Android Gradle Plugin 8.0+.
