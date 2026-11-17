# SmartCoach Recipes — Shared Conventions

Read this **before** any capability recipe (`scan.md`, `connect.md`, `measure.md`,
`settings.md`).

## How to read these recipes: contract vs. implementation

Every capability recipe contains two separable things:

- **The SDK contract** — the required call sequences, the session state machine, the
  lifecycle rules, and the gotchas. This is **architecture-independent** and is the
  actual value of these recipes. It is correct with AndroidX `ViewModel`s, MVI, plain
  Fragments/Activities, or any other architecture. The contract appears as the "Rules"
  sections and this file's "Core contract" below.
- **A reference implementation** — Kotlin that realizes the contract in an AndroidX
  **`ViewModel`** exposing `StateFlow`s. This is *one* way to house the contract, shown
  because it's the most common. In a different architecture, **do not paste the ViewModel
  code** — implement the same contract idiomatically (see "Mapping the contract to other
  architectures" below). The invariants never change; only where the state lives and
  which scope runs the coroutines.

If the target app uses `ViewModel`s, follow the reference implementation directly.
Otherwise, take the contract and map it.

---

## Step 1 — Inspect the target first (every recipe)

Do all of this before writing or prompting:

1. **Detect the architecture.** Look at how existing screens are built:
   - `androidx.lifecycle.ViewModel` subclasses (often `@HiltViewModel`) exposing
     `StateFlow`/`LiveData` → **MVVM** (use the reference implementation directly).
   - One immutable `UiState` per screen plus an intent/event function or reducer →
     **MVI / UDF**.
   - Logic in `Activity`/`Fragment` classes, no ViewModels → **Views / MVC**.
   - An app-wide `@Singleton` repository or manager that owns device state, with screens
     reading from it → **repository-owned** state.
   If it isn't MVVM, plan to map the contract (see below), not paste the ViewModel code.
2. **Audit existing SmartCoach usage.** Search the whole codebase for `SmartCoach.`,
   `sessionStateStream`, `startScanning`, `startMeasuring`, `SmartCoachSessionState`,
   `import com.pocketradar.smartcoach`. A consumer may have a prior, partial, or
   non-canonical attempt (polling `currentSessionState()` on a timer, collecting in a
   Composable, a singleton holding devices, scattered duplicate calls). Decide explicitly:
   - **Adopt** — the existing integration already matches the contract → extend it.
   - **Refactor** — it works but diverges from the contract (e.g. polling instead of
     the stream, a second collector) → migrate it to the contract, don't add beside it.
   - **Reconcile** — it's broken/half-done → repair to the contract, removing the dead
     or conflicting pieces.
   Never blindly add a fresh integration alongside an existing one — that reintroduces
   duplicate observation and conflicting state.
3. **Match local conventions** — `StateFlow` vs `LiveData`, one `UiState` data class vs
   several flows, the error surface (`errorMessage`/an error event/a sealed `UiState`),
   dependency injection (Hilt/Koin/none), naming. Reuse what's there.
4. **Scope** — where do this app's coroutines run (`viewModelScope`, a repository's
   injected `CoroutineScope`, `lifecycleScope`)? SDK calls go there.
5. **UI collection** — how does the UI read state (`collectAsStateWithLifecycle()` in
   Compose, `repeatOnLifecycle` in Views, `observe` for `LiveData`)?

---

## Core contract (architecture-neutral — applies everywhere)

These hold no matter how the app is structured:

- **Collect `SmartCoach.sessionStateStream()`; do not poll `currentSessionState()`.**
  It is a `StateFlow`: `currentSessionState()` (or `.value`) is for a one-shot read, not
  for tracking changes.
- **Every SmartCoach function is main-safe** and most are `suspend`. Call them from any
  coroutine; never move them to `Dispatchers.IO`.
- **Every failure is a thrown `SmartCoachError`.** Catch `SmartCoachError` and branch on
  `e.code` (a `SmartCoachErrorCode`); `e.message` is a readable description. Don't catch
  `Exception` broadly — it would swallow coroutine cancellation.
- **Entitlements gate scanning, connecting and measuring.** The API key decides which
  radar models the app may use. A call it doesn't allow throws `featureNotAvailable`;
  entitlements that can't be fetched or verified throw `invalidEntitlements`. Neither is
  fixed by retrying in a loop: show the message, and for `invalidEntitlements` let the user
  retry once they have network access.
- **Results arrive via the state stream, not return values.** Actions start work; the
  outcome shows up as a later `SmartCoachSessionState`.
- **The session follows the radar.** The radar can start or stop measuring on its own
  (its trigger); the session then moves between `connected` and `measuring` without
  any app call, and a measurement flow in progress completes when the radar stops.
  Handle `measuring` in the `when` even if the app never calls `startMeasuring()`.
- **Never stop collecting on a state-carried error.** `disconnected(error)` is data —
  surface the error and keep collecting. Ending the collector permanently blinds the app
  to later states.
- **One collector, one place.** Exactly one active collector of `sessionStateStream()`
  per state holder; route all state handling through its single `when`.
- **Tie the collector to its owner's scope** — `viewModelScope` in a ViewModel, the
  injected application scope in a singleton. It then stops with its owner; no manual
  teardown, no `onCleared()` cleanup for it.

Per-capability contract additions (the `connected` precondition for measuring, the
"`connect()` returning ≠ ready" rule, etc.) live in each recipe's "Rules" section.

---

## Reference implementation (ViewModel)

The base type, before any capability is added. Capability recipes add state, `when`
branches, and action functions to this.

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.pocketradar.smartcoach.SmartCoach
import com.pocketradar.smartcoach.SmartCoachError
import com.pocketradar.smartcoach.SmartCoachSessionState
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class RadarViewModel : ViewModel() {

    /** The SDK session. Throws notConfigured if SmartCoach.configure() hasn't run at launch. */
    val sessionState: StateFlow<SmartCoachSessionState> = SmartCoach.sessionStateStream()

    private val _errorMessage = MutableStateFlow<String?>(null)
    val errorMessage: StateFlow<String?> = _errorMessage.asStateFlow()

    init {
        viewModelScope.launch {
            sessionState.collect { state ->
                when (state) {
                    is SmartCoachSessionState.disconnected ->
                        state.error?.let { _errorMessage.value = it.message }
                    else -> Unit
                }
                // Capability recipes add branches and per-state work here.
            }
        }
    }

    fun dismissError() {
        _errorMessage.value = null
    }

    /** Runs an SDK call in viewModelScope, reporting a failure through errorMessage. */
    private fun launchCall(failure: String, call: suspend () -> Unit) {
        viewModelScope.launch {
            try {
                call()
            } catch (e: SmartCoachError) {
                _errorMessage.value = "$failure: ${e.message}"
            }
        }
    }
}
```

UI (Compose reference):

```kotlin
val sessionState by viewModel.sessionState.collectAsStateWithLifecycle()
val errorMessage by viewModel.errorMessage.collectAsStateWithLifecycle()
```

(`collectAsStateWithLifecycle` is in `androidx.lifecycle:lifecycle-runtime-compose`.)
The SDK keeps the session alive in its own service, so the ViewModel being cleared only
stops this screen's observation — it does not disconnect the radar.

`LiveData` variant: expose `sessionState.asLiveData()` and `errorMessage.asLiveData()`
instead; the collector in `init` is unchanged. Hilt variant: annotate
`@HiltViewModel class RadarViewModel @Inject constructor() : ViewModel()`.

---

## Mapping the contract to other architectures

Same contract, different home for three roles — **state**, **the collector**, and
**actions**. Implement idiomatically; keep every invariant above.

- **MVI / UDF** (one `UiState` + intents): fold `sessionState` and capability state into
  the `UiState` data class; the single collector maps each `SmartCoachSessionState` into a
  `UiState` update (`_uiState.update { it.copy(...) }`); actions are intents handled by
  launching the SDK call in `viewModelScope`.
- **Repository-owned state** (an app with several screens using the radar): make one
  `@Singleton` repository the **only** collector of `sessionStateStream()`, running in an
  application-lifetime `CoroutineScope` (`SupervisorJob() + Dispatchers.Main`); it
  exposes the app's own state flows and action functions. ViewModels read the repository,
  never the SDK. This keeps "one collector" true across the whole app.
- **Views / Fragments without ViewModels**: hold state in the Fragment/Activity; collect
  with `viewLifecycleOwner.lifecycleScope.launch { repeatOnLifecycle(Lifecycle.State.STARTED) { … } }`;
  actions launch in `lifecycleScope`. Prefer adding a ViewModel if the app already uses
  any — state held in a Fragment is lost on rotation.
- **A plain class with its own scope**: create `CoroutineScope(SupervisorJob() + Dispatchers.Main)`,
  run the collector and actions in it, and `cancel()` it in the class's existing teardown.

If the architecture isn't listed, map the same three roles into its idioms. When in
doubt about *what* to do, the contract governs; only the *where* is architecture-specific.

---

## Composability (any architecture)

When adding a capability to code that already collects the stream:

- Add the capability's **state**.
- Add/extend its **branches** in the single existing `when` (or the MVI reducer, etc.).
- Add its **actions**.
- Honor **cross-capability cleanup** — some recipes require work in *other* capabilities'
  branches (e.g. `measure` cancels its readings job in the non-measuring branches). Each
  recipe's "Composability" section spells out exactly what to add where.

Never add a second collector of `sessionStateStream()`.

---

## Verify

- The project builds.
- Live behavior (discovery, connection, measurement) needs a **real phone** (the Android
  emulator has no Bluetooth radio), Bluetooth on, the runtime permissions granted, and a
  configured SDK with a valid API key. A build success confirms wiring; live results need
  hardware.
