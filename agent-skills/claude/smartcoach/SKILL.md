---
name: smartcoach-sdk
description: >-
  Integrate the PocketRadar SmartCoach Android SDK into a Kotlin Android app. Use whenever
  the user asks to set up or configure the SmartCoach SDK (Gradle dependency, API key,
  Bluetooth runtime permissions, configure() at launch) or to add SmartCoach device
  scanning, connecting, measurement streaming, radar settings (units, speed range,
  sensitivity), or session-state handling — whether editing an existing ViewModel or
  scaffolding a new screen. Ensures the generated code follows the SDK's required call
  sequences and lifecycle instead of improvising the API.
---

# SmartCoach SDK integration

Adds SmartCoach SDK capabilities to a consumer's Android app by editing their code. This
skill supplies the SDK-correct behavior; you supply the scaffolding and the UI.

## Ownership boundary

- **This skill owns** the SDK specifics: which APIs to call, in what order, the
  session-state observation, cancellation, and error handling. Follow the recipes exactly
  for these — do NOT write SmartCoach API calls from memory.
- **You own** everything around them: the app's architecture, locating or creating the
  state holder, building the UI (Compose or Views), and matching the project's
  conventions and style.

## Contract vs. implementation

Each recipe states an **SDK contract** (architecture-neutral rules — the "Rules"
sections) plus a **reference implementation** (Kotlin in an AndroidX `ViewModel`). The
contract holds in any architecture (MVVM, MVI, Views/Fragments, a plain class with its own
`CoroutineScope`). If the target app uses `ViewModel`s, use the reference code directly.
If it does not, implement the same contract idiomatically — `conventions.md` has a
"Mapping the contract to other architectures" section.

## How to use

1. For the capability recipes, **always read `recipes/conventions.md` first.** It
   defines the shared ViewModel structure they build on.
2. Then read the recipe(s) for what the user wants:
   - **SDK setup / bootstrap** (Gradle dependency, API key, runtime permissions,
     `configure()` at launch) → `recipes/configure.md` — standalone; does not use
     `conventions.md`. It prompts the user for their API key.
   - **Device scanning** → `recipes/scan.md`
   - **Connect / disconnect** → `recipes/connect.md`
   - **Stream measurements** → `recipes/measure.md`
   - **Radar settings** (units, speed range, sensitivity) → `recipes/settings.md`
3. For a request spanning capabilities ("connect and measure", "a full scan→connect→
   measure screen"), apply the recipes in order into **one** ViewModel — they share a
   single session-state collector by design (see each recipe's "Composability").
4. Apply into the state holder the user names. For an abstract request ("build a
   screen that scans"), first scaffold it in the project's architecture and style, then
   apply the recipe(s).

Before writing any code, do `conventions.md` Step 1: **detect the app's architecture**
(don't assume MVVM) and **audit existing SmartCoach usage** (a partial or non-canonical
prior attempt must be adopted/refactored/reconciled, never duplicated alongside).

## Prerequisites (assumed by every capability recipe)

- The app depends on `com.pocketradar:smartcoach`.
- `SmartCoach.configure()` is called once at launch, in `Application.onCreate()`.
- The app requests the Bluetooth runtime permissions before scanning.

If any is missing, apply `recipes/configure.md` first (or offer to) — SDK calls fail with
`notConfigured` or `permissionDenied` otherwise.

## Non-negotiable rules (all recipes)

- Every `SmartCoach` function is main-safe: call it from any coroutine. Never wrap SDK
  calls in `withContext(Dispatchers.IO)`.
- Every failure is a thrown `SmartCoachError`; branch on `e.code` (`SmartCoachErrorCode`).
- Results arrive through `SmartCoach.sessionStateStream()`, not as return values.
- The session follows the radar: `measuring` can appear without the app calling
  `startMeasuring()` (the radar's own trigger). Handle it; never treat it as an error.
- Session observation is **one** collector of `sessionStateStream()`, launched in
  `viewModelScope` from the ViewModel's `init` (it cancels with the ViewModel — no manual
  teardown). The UI collects the ViewModel's flows with `collectAsStateWithLifecycle()`.
- **Never stop collecting on a state-carried error** — surface `disconnected(error)` via
  the error state and keep collecting.
- **Compose, don't duplicate:** if the ViewModel already collects the session stream,
  extend its existing `when` — never add a second collector.
