# Android review reference (2026)

Apply to Kotlin/Compose/Gradle files. Only review lines present in the diff (`+` lines). Never flag pre-existing code.

## Table of contents

1. [Deprecations & platform changes (2026)](#deprecations--platform-changes-2026)
2. [Architecture & MVVM/MVI](#architecture--mvvmmvi)
3. [Jetpack Compose](#jetpack-compose)
4. [Coroutines & Flows](#coroutines--flows)
5. [Lifecycle awareness](#lifecycle-awareness)
6. [Dependency injection](#dependency-injection-dagger--hilt)
7. [Security](#security)
8. [Performance & memory](#performance--memory)
9. [Testing](#testing)
10. [Kotlin quality](#kotlin-quality)
11. [Navigation](#navigation)
12. [Resources & localisation](#resources--localisation)
13. [Accessibility](#accessibility)
14. [Dependency / build hygiene](#dependency--build-hygiene)

---

## Deprecations & platform changes (2026)

Flag **newly added** usage of anything below. Context: Google Play requires new apps/updates to target **API 36 (Android 16)** from Aug 31, 2026; API 37 planned for Aug 2027.

### Android 16 / API 36 behavior changes & deprecations

| Newly added usage of… | Status | Replacement / note | Severity |
|---|---|---|---|
| `screenOrientation` manifest locks, `setRequestedOrientation()`, `resizeableActivity=false`, min/max aspect-ratio restrictions | Being phased out; **ignored on large screens (sw ≥ 600dp) when targeting API 36+**; no opt-out at API 37 | Build adaptive layouts (`WindowSizeClass`, canonical layouts) instead of locking orientation | 🟠 |
| `announceForAccessibility()` | Deprecated | Use semantics-based live regions / `AccessibilityNodeInfo` alternatives per the API docs | 🟡 |
| `AccessibilityNodeInfo.setLabeledBy` / `getLabeledBy` | Deprecated in API 36 | `addLabeledBy` / `removeLabeledBy` / `getLabeledByList` | 🟡 |
| `elegantTextHeight` attribute | Deprecated; **ignored when targeting API 36** | Remove; verify layouts for Arabic/Thai/Tamil etc. tall scripts | 🟡 |
| Opting out of edge-to-edge (`windowOptOutEdgeToEdge`, legacy status-bar color APIs) | Edge-to-edge is enforced for apps targeting API 36; opt-out removed | `enableEdgeToEdge()` + `windowInsetsPadding` / `WindowInsets` APIs | 🟠 |
| `PendingIntent` without `FLAG_IMMUTABLE`/`FLAG_MUTABLE` | Crash on API 31+ | Add `PendingIntent.FLAG_IMMUTABLE` unless mutability is required | 🔴 |
| Broad implicit intents relying on lenient resolution | API 36 tightens implicit-intent resolution | Prefer explicit intents; validate receiving component | 🟠 |
| Writable-then-executed dynamically loaded code (DEX/native) | API 36 requires files marked read-only before execution | `setReadOnly()` before loading | 🔴 |
| `AsyncTask`, `Loader`s, `IntentService`, `Handler()` no-arg ctor | Long deprecated | Coroutines/`WorkManager`, `Handler(Looper.getMainLooper())` | 🟡 |
| `startActivityForResult` / `onActivityResult`, `requestPermissions` overrides | Deprecated | Activity Result APIs (`registerForActivityResult`) | 🟡 |
| `SharedPreferences` for new preference storage | Superseded | Jetpack DataStore (Preferences or Proto) | 🟡 |
| `kapt` for new annotation processors | Maintenance mode | KSP | 🟡 |
| `LiveData` in new code | Superseded | `StateFlow`/`SharedFlow` (+ `asLiveData()` only at legacy boundaries) | 🟡 |
| `android.preference.*`, `PreferenceManager` | Deprecated | AndroidX Preference / DataStore | 🟡 |
| Back handling via `onBackPressed()` override | Deprecated | `OnBackPressedDispatcher` / predictive back (`android:enableOnBackInvokedCallback`) — Android 16 extends predictive back to 3-button nav | 🟠 |

### Compose & AndroidX supersessions

| Newly added usage of… | Status | Replacement | Severity |
|---|---|---|---|
| `collectAsState()` on Android UI flows | Superseded | `collectAsStateWithLifecycle()` (lifecycle-aware; stops collection in background) | 🟡 |
| Material 2 (`androidx.compose.material.*`) components in new UI | Superseded | Material 3 (`androidx.compose.material3.*`) equivalents | 🟡 |
| Accompanist libraries that were upstreamed (SystemUiController, Pager, FlowLayout, Navigation-Material, Permissions where applicable, Placeholder…) | Deprecated/archived | Core Compose equivalents: `enableEdgeToEdge()`+insets, `androidx.compose.foundation.pager`, `FlowRow`/`FlowColumn`, `material-navigation`, etc. | 🟡 |
| String-route Navigation-Compose in new graphs | Superseded | Type-safe navigation (serializable route objects) or the project's typed-nav wrapper | 🟡 |
| `LocalLifecycleOwner` from `androidx.compose.ui.platform` | Moved | Import from `androidx.lifecycle.compose` | 🟢 |
| `rememberRipple()` | Deprecated | `ripple()` indication API | 🟢 |

If the diff uses an API you don't recognize or you're unsure whether it's been deprecated since this file was written, search the web before commenting.

---

## Architecture & MVVM/MVI

- ViewModel holds no reference to `Activity`, `Fragment`, `View`, or `Context` (use `@ApplicationContext` via Hilt if truly needed)
- UI state lives in `StateFlow`/`MutableStateFlow` in the ViewModel, not in `Activity`/`Fragment` fields
- Side effects flow through a `SharedFlow` or sealed `Effect` channel — never called directly from the ViewModel into UI
- `viewModelScope` used for ViewModel work; `lifecycleScope` / `repeatOnLifecycle` used for UI collection — never `GlobalScope`
- No business logic inside Composable functions or `Fragment`s; logic belongs in ViewModel or domain layer
- No direct repository/data-layer access from UI layer; routes through ViewModel
- `SavedStateHandle` used for state that must survive process death (navigation arguments, form input)
- Unidirectional data flow preserved: state flows down, events flow up — flag two-way mutable state passed deep into the tree

## Jetpack Compose

- **Recomposition stability**: lambdas passed to composables are stable (method references, `remember`ed, or created outside hot paths) to avoid recomposition on every call-site recomposition
- Types passed to composables are primitive, `@Stable`, `@Immutable`, or data classes of stable types — stdlib `List`/`Map` should be `ImmutableList`/`ImmutableMap` (kotlinx.collections.immutable) or wrapped
- `remember { }` for objects expensive to create that should survive recomposition; `rememberSaveable { }` for values that must survive configuration changes
- Side-effect APIs used correctly:
  - `LaunchedEffect(key)` for coroutine work tied to a lifecycle event or key change
  - `DisposableEffect` for resources that must be cleaned up (listeners, subscriptions)
  - `SideEffect` only for non-suspending synchronous side effects after every successful recomposition
  - No coroutine launches or side effects directly in a composable body outside these APIs
- `items(key = ...)` / `key()` used in `LazyColumn`/`LazyRow` to preserve item state on list changes
- Scaffold `paddingValues` from the content lambda actually applied — content otherwise renders under system bars
- `Modifier` chains ordered correctly: layout modifiers (size, padding) before drawing modifiers (background, border)
- State hoisted: composables receive state + event lambdas; no `MutableState` mutated from outside its owning scope
- `derivedStateOf { }` for computed values that depend on other state, to avoid unnecessary recompositions
- `collectAsStateWithLifecycle()` for flow collection (see deprecation table)
- New public composables have a `@Preview` (or the project's screenshot-test equivalent)
- No expensive work (I/O, parsing, sorting large lists) in the composition body — precompute in the ViewModel or `remember`

## Coroutines & Flows

- No `runBlocking` on the main thread; no blocking I/O on `Dispatchers.Main`
- `launch`/`async` always structured — no orphaned coroutines or fire-and-forget outside a meaningful scope
- `launch { }` blocks that can throw use `try/catch` or a `CoroutineExceptionHandler`
- `collect` vs `collectLatest`: `collectLatest` when only the latest emission matters and previous work should be cancelled (search, debounce); `collect` when every emission must be processed
- `StateFlow` for state (single current value, replayed); `SharedFlow` for one-shot events
- `flow { }` builders don't capture an outer coroutine scope — use `callbackFlow` for callback-based APIs
- `flowOn(Dispatchers.IO)` applied in the data layer, not repeated in the ViewModel; inject dispatchers rather than hardcoding for testability
- `combine`, `zip`, `flatMapLatest` chosen for the intended semantics — verify the operator matches
- No `Flow` collected inside another `Flow` builder (use `flatMapLatest`/`flatMapMerge`)
- `CancellationException` never swallowed by a broad `catch (e: Exception)` — rethrow it

## Lifecycle awareness

- `Activity`/`Fragment` context never stored in ViewModel, companion object, or app-lifetime singleton
- `repeatOnLifecycle(Lifecycle.State.STARTED)` when collecting flows from Views/Fragments
- `viewLifecycleOwner` (not `this`) when observing in fragments
- Listeners/callbacks/subscriptions set in `onStart`/`onResume` removed in the paired `onStop`/`onPause`
- `onSaveInstanceState` / `SavedStateHandle` for UI state that should survive rotation and process death

## Dependency injection (Dagger / Hilt)

- Correct scope: `@Singleton` app-scoped, `@ActivityRetainedScoped` ViewModel-lifetime, `@ViewModelScoped` per-ViewModel
- `@ApplicationContext` instead of raw `Context` in singletons
- No manual `DaggerXxxComponent.create()` in production code — Hilt entry points
- `@InstallIn` target matches the intended lifetime
- `@Binds` preferred over `@Provides` for interface-to-implementation binding

## Security

- No API keys, tokens, passwords, or secrets in source files, string resources, or `local.properties` committed to the repo
- `WebView`: `setJavaScriptEnabled(false)` unless required; `setAllowFileAccess(false)` unless required; no `addJavascriptInterface` with untrusted content
- External-component intents are explicit or validate the receiving app
- No tokens/PII in `Log.*` — debug-only logging guarded by `BuildConfig.DEBUG`
- Sensitive data not in plain `SharedPreferences` — `EncryptedSharedPreferences` or encrypted DataStore
- Deep link URIs validated before acting on host/path/query parameters
- Exported components (`android:exported="true"`) justified; new manifest entries default to non-exported
- Certificate pinning / network-security-config changes reviewed deliberately — never a blanket `cleartextTrafficPermitted="true"`

## Performance & memory

- No static references to `Activity`, `Fragment`, `View`, or `Context` subclass
- Bitmaps managed via Coil/Glide; no manual large-bitmap allocations
- `RecyclerView`/`LazyList`: `DiffUtil` or stable keys — no `notifyDataSetChanged()` except where documented
- Heavy computation off the main thread — `Dispatchers.Default`/`IO`
- No object allocations inside `draw()`/`onDraw()` — pre-allocate
- Overly frequent work in tight loops (JSON parsing, regex compilation) hoisted out
- `ViewModel` caches data that shouldn't be re-fetched on recomposition/rotation

## Testing

- New public/internal logic has corresponding unit tests
- ViewModel tests use `StandardTestDispatcher` + `TestCoroutineScheduler` (or `UnconfinedTestDispatcher` for simple cases) — never `Dispatchers.Main` directly; use a `MainDispatcherRule`
- Turbine (or `runTest { }`) for `Flow` emissions — no manual `take(1).toList()` patterns
- MockK for mocking (Mockito acceptable if already in use)
- Arrange / Act / Assert structure with clear separation
- Edge cases covered: empty list, null input, network error, cancellation
- No `Thread.sleep()` — `advanceTimeBy` / `runTest`
- **Source set placement**: Paparazzi/screenshot/snapshot tests run on the JVM and must live in `src/test/`, not `src/androidTest/`. Check the path of every new test file.
- **Project conventions**: before approving a new test file's structure, scan existing tests for shared base classes or naming patterns (e.g. `find . -name '*Test.kt' -path '*/test/*' | head -10`). Flag new tests that skip an established base class (e.g. `PaparazziSnapshotBaseTest`) or its helpers when the rest of the codebase uses them.
- **Tests must exercise the code under test**: verify each test fires the event / calls the function it claims to test. A test that constructs state locally but never passes it to the ViewModel, or fires the wrong event, gives false confidence — flag it even if it passes.

## Kotlin quality

- `sealed` when variants carry different data; `enum` for simple constant sets
- `data class` for value types needing `equals`/`hashCode`/`copy` — not for classes with identity semantics or mutable state
- Extension functions don't break encapsulation
- `object` singletons not used for stateful classes that need testing or DI
- Null safety: no new `!!` — `?: return` / `?: error(...)` / `requireNotNull` with a message
- `when` on sealed types exhaustive — no trailing `else ->` that swallows future variants
- `inline`/`reified` where appropriate for generic utilities
- Scope functions (`apply`, `let`, `run`, `also`, `with`) used idiomatically, not chained to obscure intent
- **No FQN in place of imports**: inline fully-qualified names (`androidx.compose.runtime.remember(...)`) replaced with a top-level import — implementation, test, and preview files alike
- **Nullable parameters carry `= null` defaults** when always optional at every call site
- **`data class` with empty `{ }` body** — remove the body

## Navigation

- Deep link URIs validated before acting on parameters
- Back-stack manipulation (`popUpTo`, `launchSingleTop`) intentional — comment if non-obvious
- `NavController` not stored in ViewModel — navigation events emitted as effects, consumed in UI
- Type-safe args between destinations (see deprecation table for string routes)

## Resources & localisation

- No hardcoded user-visible strings — `strings.xml` with meaningful keys
- No hardcoded dimensions — `dimens.xml`, design tokens, or design-system `Dp` values
- No hardcoded colors outside a theme/design-system file
- Plurals via `plurals` resource, not `if (count == 1)` inline
- RTL: `start`/`end` not `left`/`right` in XML; `Arrangement.Start`/`End` in Compose
- Format-specifier escaping correct (`%%` for literal percent, positional args `%1$s` when multiple)

## Accessibility

- Interactive elements have `contentDescription` or `semantics { }`
- Decorative images: `contentDescription = null`
- Stateful components (toggle, checkbox, selection) expose `stateDescription`
- `Modifier.semantics(mergeDescendants = true)` groups related content
- Touch targets ≥ 48×48dp
- No `announceForAccessibility` in new code (see deprecation table)

## Dependency / build hygiene

- New dependencies go in the version catalog (`libs.versions.toml`) — not hardcoded in `build.gradle.kts`
- No duplicates already provided transitively
- ProGuard/R8 keep rules added for any reflection-heavy or serialization library introduced in this PR
- `isShrinkResources`/minification settings not weakened without justification
- New modules follow the project's convention plugins; AGP 9-era source-set config uses `kotlin.srcDir()` for Kotlin sources
- No `-SNAPSHOT` or dynamic (`+`) versions introduced
