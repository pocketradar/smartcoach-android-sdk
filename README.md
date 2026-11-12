# Pocket Radar SmartCoach Android SDK

⚠️ **IMPORTANT**

This repository contains the **Pocket Radar SmartCoach Software Development Kit (SDK)** used to integrate Pocket Radar hardware into third-party Android applications.

The SDK is **proprietary software owned by Pocket Radar, Inc.**

This SDK is **NOT open source**.

Access to this repository does **not grant permission to use the SDK**.

Use of the SDK requires:

- Approval from Pocket Radar
- Issuance of a valid **Pocket Radar API key**
- A **separate commercial or partner agreement**

Applications using the SDK will not function without a valid API key issued by Pocket Radar.

---

## Overview

The Pocket Radar SmartCoach SDK enables approved partners to integrate Pocket Radar hardware into their Android applications. The SDK enables applications to discover, connect to, and stream measurement data from SmartCoach radar devices.

Typical integrations include:

- Sports performance platforms
- Training applications
- Analytics platforms
- Partner software products

The Android SDK has the same API as the [iOS SDK](https://github.com/pocketradar/smartcoach-ios-sdk), name for name, so one integration design covers both platforms.

> **This SDK is currently in beta.** APIs may change between releases. Please do not publish apps using the beta SDK to Google Play.

## Requirements

- Android 10 (API 29) or later
- `compileSdk` 36 or later
- Kotlin 2.1 or later
- Android Gradle Plugin 8.0 or later

## Installation

### Gradle

Add the SmartCoach Maven repository in `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://pocketradar.github.io/smartcoach-android-sdk/maven") }
    }
}
```

Add the dependency in your app module's `build.gradle.kts`, using the latest version from [CHANGELOG.md](CHANGELOG.md):

```kotlin
dependencies {
    implementation("com.pocketradar:smartcoach:<version>")
}
```

## Configuration

### API Key

Add your API key to your app's `AndroidManifest.xml`, inside `<application>`:

```xml
<meta-data
    android:name="com.pocketradar.smartcoach.API_KEY"
    android:value="YOUR_API_KEY_HERE" />
```

### Bluetooth Permissions

The SDK declares the Bluetooth, location and foreground-service permissions it needs in its own manifest; they merge into your app automatically. Android also requires your app to **request** some of them at runtime before scanning:

| Android version | Request at runtime |
|---|---|
| 12 (API 31) and later | `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` |
| 10–11 (API 29–30) | `ACCESS_FINE_LOCATION` |
| 13 (API 33) and later, recommended | `POST_NOTIFICATIONS`, so the connection notification shows |

SDK calls made without them throw `SmartCoachError` with code `permissionDenied`.

### Foreground Service

While a radar session is active, the SDK runs a foreground service of type `connectedDevice` and shows an ongoing notification, which keeps the connection alive when your app is in the background. Customize its text, channel and icon with `SmartCoachDeviceConfigurationOptions.notification`.

## Quick Start

```kotlin
import android.app.Application
import android.util.Log
import com.pocketradar.smartcoach.SmartCoach
import com.pocketradar.smartcoach.SmartCoachError

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        try {
            SmartCoach.configure(this)
        } catch (e: SmartCoachError) {
            Log.e("MyApp", "SmartCoach configuration failed: ${e.message}")
        }
    }
}
```

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.pocketradar.smartcoach.SmartCoach
import com.pocketradar.smartcoach.SmartCoachError
import com.pocketradar.smartcoach.SmartCoachRadar
import com.pocketradar.smartcoach.SmartCoachSessionState
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class ScanningViewModel : ViewModel() {
    /** The SDK's session: collect it in the UI with collectAsStateWithLifecycle(). */
    val sessionState: StateFlow<SmartCoachSessionState> = SmartCoach.sessionStateStream()

    val availableDevices = MutableStateFlow<List<SmartCoachRadar>>(emptyList())
    val errorMessage = MutableStateFlow<String?>(null)

    init {
        viewModelScope.launch {
            sessionState.collect { state ->
                when (state) {
                    is SmartCoachSessionState.scanning -> availableDevices.value = state.radars
                    is SmartCoachSessionState.disconnected -> {
                        availableDevices.value = emptyList()
                        // Keep collecting: errors arrive as data on the stream.
                        state.error?.let { errorMessage.value = it.message }
                    }
                    else -> availableDevices.value = emptyList()
                }
            }
        }
    }

    fun startScanning() {
        if (sessionState.value is SmartCoachSessionState.scanning) return
        viewModelScope.launch {
            try {
                SmartCoach.startScanning()
            } catch (e: SmartCoachError) {
                errorMessage.value = "Failed to start scan: ${e.message}"
            }
        }
    }

    fun stopScanning() {
        viewModelScope.launch {
            try {
                SmartCoach.stopScanning()
            } catch (e: SmartCoachError) {
                errorMessage.value = "Failed to stop scan: ${e.message}"
            }
        }
    }
}
```

Devices are discovered, connected, and measured through the **session state stream**: SDK calls start work, and results arrive as states. Every SDK function is main-safe and throws `SmartCoachError`.

## What's New

See [CHANGELOG.md](CHANGELOG.md) for behaviour changes and additions in each release.

## Documentation

The full API reference for each release is published at:

**https://pocketradar.github.io/smartcoach-android-sdk/docs/&lt;version&gt;/**

It covers every public type and function, a quick start, and the differences from the iOS SDK. Android Studio also shows the same documentation inline (Quick Documentation, `F1` / `Ctrl+Q`).

## Sample Project

A complete sample app demonstrating SDK integration is available in a separate repository:

**[SmartCoach Android SDK Sample App](https://github.com/pocketradar/android-smartcoach-sdk-sample)**

The sample app shows runtime permissions, device discovery, connection management, radar settings, and streaming measurement data.

## AI-Assisted Integration (Agent Skills)

This repository ships task-scoped recipes that let an AI coding assistant integrate
specific SDK capabilities into your app — "set up the SmartCoach SDK in this app,"
"add device scanning to this view model," "build a screen that connects and streams
speeds" — following the SDK's required call sequences instead of improvising them.

**Supported today: Claude (Claude Code).** The underlying recipes
([`agent-skills/recipes/`](agent-skills/recipes/)) are vendor-neutral markdown, so
other assistants (Codex, Gemini, Cursor) can use them directly — point your agent at
the relevant recipe file.

### Install (Claude Code)

Clone this repository at the tag matching your SDK version and run the installer:

```bash
git clone --depth 1 --branch <release-tag> https://github.com/pocketradar/smartcoach-android-sdk
cd smartcoach-android-sdk/agent-skills
./install.sh                          # user-level: available in all your projects
# or:
./install.sh /path/to/your/app        # project-level: committed with one app
```

Then start a fresh Claude Code session in your project and ask naturally, e.g.
*"Set up the SmartCoach SDK in this app"* or *"Add SmartCoach device scanning to
DevicesViewModel."*

> The recipes are version-locked to SDK behavior — use the same release tag as the
> SDK version in your app, and re-run `install.sh` after updating the SDK.

See [`agent-skills/README.md`](agent-skills/README.md) for details.

## Feedback & Reporting Issues

Your feedback during the beta is invaluable — please use [GitHub Issues](https://github.com/pocketradar/smartcoach-android-sdk/issues) to report bugs, share feedback on API design, or request features. When filing a bug, include:

- SDK version
- Android version and device model
- Steps to reproduce
- Expected vs actual behavior
- Relevant logs or code snippets (set `debugLogging = true` in `SmartCoachDeviceConfigurationOptions` for more detail)

For urgent support, contact [partners@pocketradar.com](mailto:partners@pocketradar.com).

## Partner Access

API keys are issued only to **approved Pocket Radar partners**.

If you are interested in integrating Pocket Radar technology into your application, please contact:

**partners@pocketradar.com**

Pocket Radar will work with partners to establish a commercial agreement defining:

- SDK usage rights
- Distribution rights
- Support
- Commercial terms

---

## License

Use of this SDK is governed by the license terms in the **LICENSE** file in this repository.
The SDK is proprietary software and is not distributed under an open source license.

See [LICENSE](LICENSE) for complete details.
