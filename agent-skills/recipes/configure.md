# Recipe: Configure the SDK (app bootstrap)

Set up a consumer app to use the SmartCoach SDK: the Gradle dependency, the API key in the
manifest, a `SmartCoach.configure()` call at launch, and the Bluetooth runtime permission
request.

This recipe is different from the capability recipes (`scan`/`connect`/`measure`/
`settings`): it edits build files, the manifest and app bootstrap code, not a ViewModel,
and it does not use `conventions.md`'s ViewModel structure.

**Boundary: change only what this recipe names.** Add the repository, the dependency, the
manifest entry and the bootstrap code; do not upgrade the Android Gradle Plugin, Kotlin,
or other dependencies, and do not raise `minSdk` or `compileSdk` without the user's
explicit agreement (Step 2). Build-tool upgrades ripple through the whole app and are the
user's decision.

---

## Step 1 — Inspect the project (do not skip)

Gather these facts before prompting or editing:

1. **Build script language and layout.** `settings.gradle.kts`/`build.gradle.kts`
   (Kotlin DSL) or `settings.gradle`/`build.gradle` (Groovy). Find the app module
   (the one applying `com.android.application`).
2. **Version catalog?** If `gradle/libs.versions.toml` exists and the app module uses
   `libs.` aliases, add the SDK there, not as a hard-coded string.
3. **Is the SDK already present?** Search build files for `com.pocketradar:smartcoach` and
   `pocketradar.github.io`. If present, this recipe becomes "verify and fix".
4. **SDK requirements.** Read the app module's `minSdk` and `compileSdk`, and the Kotlin
   and Android Gradle Plugin versions. The SDK needs `minSdk` 29+, `compileSdk` 36+,
   Kotlin 2.1+, AGP 8.0+.
5. **Application class.** In `AndroidManifest.xml`, does `<application>` have an
   `android:name`? If so, that class's `onCreate()` is where `configure()` goes. Also search
   for `SmartCoach.configure` — if it's already called somewhere, verify it rather than
   adding a second call.
6. **Existing permission handling.** Search for `BLUETOOTH_SCAN`,
   `RequestMultiplePermissions`, `checkSelfPermission`. Reuse an existing permission flow
   rather than adding a parallel one.
7. **UI toolkit.** Compose (`setContent {`) or Views — this decides the permission
   request snippet in Step 6.

## Step 2 — Prompt the user

Ask for, in one exchange:

1. **The SmartCoach API key.** Validate it: it must parse as a UUID
   (`java.util.UUID.fromString` succeeds — 8-4-4-4-12 hex). If it doesn't, say so and ask
   again. If the user doesn't have one yet, use the placeholder
   `00000000-0000-0000-0000-000000000000`, and clearly tell them where to replace it and
   that PocketRadar partners get keys from partners@pocketradar.com.
2. **Auto-reconnect?** Whether the SDK should automatically reconnect after unexpected
   connection drops (`autoReconnect = true`). Default: off.
3. **Only if Step 1.4 found a mismatch:** whether they agree to raise `minSdk` to 29 /
   `compileSdk` to 36 (or upgrade Kotlin/AGP). If they decline, stop: the SDK can't be
   added to this app as it is.

Note on key handling: the key ships readable inside the APK and gets committed with the
project. That is the expected model for this SDK (the backend enforces entitlements
server-side), but if the user's repo is public or they object, suggest supplying it
through `manifestPlaceholders` from `local.properties` or a Gradle property instead —
don't build that unprompted.

## Step 3 — Gradle

**Repository** — in `settings.gradle(.kts)`, inside
`dependencyResolutionManagement { repositories { … } }`, after `google()` and
`mavenCentral()`:

```kotlin
maven { url = uri("https://pocketradar.github.io/smartcoach-android-sdk/maven") }
```

(Groovy: `maven { url 'https://pocketradar.github.io/smartcoach-android-sdk/maven' }`.)
If the project declares repositories in the root `build.gradle` `allprojects { }` block
instead, add it there.

**Dependency** — in the app module, using the SDK version the user wants (the latest in
the SDK's CHANGELOG if they don't say):

```kotlin
dependencies {
    implementation("com.pocketradar:smartcoach:<version>")
}
```

With a version catalog: add `smartcoach = "<version>"` under `[versions]` and
`smartcoach = { group = "com.pocketradar", name = "smartcoach", version.ref = "smartcoach" }`
under `[libraries]`, then `implementation(libs.smartcoach)`.

The SDK brings `kotlinx-coroutines` with it; nothing else is required.

## Step 4 — The API key

In the app's `AndroidManifest.xml`, inside `<application>`:

```xml
<meta-data
    android:name="com.pocketradar.smartcoach.API_KEY"
    android:value="PASTE-KEY-HERE" />
```

Do **not** add Bluetooth, location or foreground-service `<uses-permission>` entries: the
SDK's manifest declares them and they merge in automatically.

## Step 5 — The `configure()` call

`configure()` must run **once, at launch, before any other SDK call**: in
`Application.onCreate()`. Pick by what Step 1.5 found:

**Existing `SmartCoach.configure` call** → verify it: runs in `Application.onCreate()`,
catches `SmartCoachError`, treats `alreadyConfigured` as benign. Fix in place; do not add
another.

**Existing Application class** → add to its `onCreate()`, after `super.onCreate()`:

```kotlin
import android.util.Log
import com.pocketradar.smartcoach.SmartCoach
import com.pocketradar.smartcoach.SmartCoachError
import com.pocketradar.smartcoach.SmartCoachErrorCode

override fun onCreate() {
    super.onCreate()
    try {
        SmartCoach.configure(this)
    } catch (e: SmartCoachError) {
        // alreadyConfigured is benign. Anything else: don't crash the app — SDK calls
        // report notConfigured until it's fixed.
        if (e.code != SmartCoachErrorCode.alreadyConfigured) {
            Log.e("SmartCoach", "Configuration failed: ${e.message}")
        }
    }
}
```

**No Application class** → create one in the app's base package (e.g. `MyApp.kt`) with
the `onCreate()` above, and register it on the manifest's `<application>` element:

```xml
<application
    android:name=".MyApp"
    … >
```

If the app uses Hilt, the existing `@HiltAndroidApp` class is the Application class —
add the call there.

**Options:** if the user wanted auto-reconnect (Step 2.2), pass options:

```kotlin
SmartCoach.configure(
    this,
    SmartCoachDeviceConfigurationOptions(autoReconnect = true),
)
```

Also available: `debugLogging = true` (useful during integration), `logger` (route SDK
logs to the app's own logging), and `notification = SmartCoachNotificationConfig(...)`
(text, channel and icon of the notification the SDK shows while connected).

## Step 6 — Runtime permissions

Android requires the app to request these at runtime; the SDK cannot. Without them,
scanning and connecting throw `SmartCoachError` with code `permissionDenied`.

| Android version | Required | Recommended |
|---|---|---|
| 12 (API 31)+ | `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` | `POST_NOTIFICATIONS` on 13 (API 33)+ |
| 10–11 (API 29–30) | `ACCESS_FINE_LOCATION` | — |

Add one helper (skip if Step 1.6 found an equivalent):

```kotlin
import android.Manifest
import android.content.Context
import android.content.pm.PackageManager
import android.os.Build
import androidx.core.content.ContextCompat

object BluetoothPermissions {
    /** Needed to scan for and connect to radars. */
    val required: List<String> = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        listOf(Manifest.permission.BLUETOOTH_SCAN, Manifest.permission.BLUETOOTH_CONNECT)
    } else {
        listOf(Manifest.permission.ACCESS_FINE_LOCATION)
    }

    /** Lets the SDK's connection notification show; the SDK works without it. */
    val optional: List<String> = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        listOf(Manifest.permission.POST_NOTIFICATIONS)
    } else {
        emptyList()
    }

    fun granted(context: Context): Boolean = required.all {
        ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED
    }
}
```

Request them before the first scan — on the screen that scans, not at app launch.

**Compose:**

```kotlin
val context = LocalContext.current
var permitted by remember { mutableStateOf(BluetoothPermissions.granted(context)) }
val launcher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions(),
) { permitted = BluetoothPermissions.granted(context) }

if (!permitted) {
    Button(onClick = {
        launcher.launch((BluetoothPermissions.required + BluetoothPermissions.optional).toTypedArray())
    }) { Text("Allow Bluetooth") }
    return
}
```

**Views (Activity/Fragment):** register in the class body (not in a click handler):

```kotlin
private val permissionLauncher =
    registerForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) {
        if (BluetoothPermissions.granted(requireContext())) startScanning()
    }

// before scanning:
if (BluetoothPermissions.granted(requireContext())) startScanning()
else permissionLauncher.launch((BluetoothPermissions.required + BluetoothPermissions.optional).toTypedArray())
```

User-facing strings (the button label, any explanation) go in `res/values/strings.xml`
following the app's existing convention.

## Step 7 — Verify

- The project builds (`./gradlew :app:assembleDebug`).
- On a real phone: launch should log no configuration failure, and the permission
  request should appear on the scanning screen. `configure()` needs no Bluetooth and works
  on the emulator; scanning and connecting need real hardware.
- Failure decoding: `missingApiKey` → the `<meta-data>` entry is missing or misnamed.
  `permissionDenied` on scan → the runtime request didn't run or was denied. A Gradle
  "Could not find com.pocketradar:smartcoach" → the repository line is missing or in the
  wrong block. "Manifest merger failed … minSdkVersion" → `minSdk` below 29.

---

## Rules baked into this recipe

- **`configure()` exactly once, in `Application.onCreate()`, before any other SDK call.**
  A second call throws `alreadyConfigured` — which the generated code treats as benign,
  never fatal.
- **Never crash the host app over configuration.** Catch and report; the SDK surfaces
  `notConfigured` on later calls, which is diagnosable.
- **The SDK declares its permissions; the app requests the runtime ones.** Never
  duplicate the SDK's `<uses-permission>` entries, and never skip the runtime request.
- **Don't upgrade the build toolchain or SDK levels without the user's agreement.**
