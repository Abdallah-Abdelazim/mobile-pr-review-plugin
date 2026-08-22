# KMP review reference

Apply to any file under `kmp/` or in multiplatform source sets. Rules are stricter than single-platform code because mistakes in `commonMain` break all platforms simultaneously.

## Source set map

| Path pattern | Meaning |
|---|---|
| `kmp/…/commonMain/…` | Common code — strictest rules |
| `kmp/…/androidMain/…` | Android actuals |
| `kmp/…/iosMain/…` | iOS/Native actuals |
| `kmp/…/commonTest/…` | Common tests |
| `kmp/…/engine-ios-bindings/…` | Swift interop layer |

Apply the Android checklist (`android.md`) additionally to `androidMain`, and the iOS checklist (`ios.md`) additionally to `iosMain`/bindings where relevant.

## Source set hygiene

- No Android, JVM, or iOS platform imports in `commonMain` — `android.*`, `java.*`, `javax.*`, `platform.*`, `UIKit`, `Foundation` are all forbidden in common code
- No `actual` implementations in `commonMain` — actuals belong in platform source sets
- New platform-specific logic added in the correct source set — not worked around with `if (Platform.isAndroid)`
- Genuinely cross-platform utility code pushed to `commonMain` rather than duplicated per platform

## expect / actual

- `expect` declarations are minimal — interface only, no business logic
- `actual` implementations don't duplicate logic that could live in `commonMain`; they delegate to the hook and call shared code
- Every `expect` has a corresponding `actual` in each required source set — a missing actual breaks the platform not being tested locally
- `actual typealias` over `actual class` where the platform type is a direct equivalent
- `expect` functions that can throw declare `@Throws` so Kotlin/Native surfaces them as `NSError` to Swift callers

## Coroutines & concurrency (KMP-safe)

- `synchronized {}` **not used in common or iOS source sets** — JVM-only, crashes on Kotlin/Native; use `kotlinx.coroutines.sync.Mutex`
- `java.util.concurrent.atomic.*` not in common code — use `kotlinx.atomicfu` (`AtomicInt`, `AtomicRef`)
- `ThreadLocal` not in common code — coroutine context elements or explicit state
- `Dispatchers.Main` only in UI-layer code, not the shared engine; the engine emits via `Flow` and lets callers choose their dispatcher
- `newSingleThreadContext` / `newFixedThreadPoolContext` not created in common code without being closed

## Serialization

- `kotlinx.serialization` (`@Serializable`, `Json { }`) in common code — not Gson, Moshi, or `NSCoding`
- `@SerialName` where the JSON key differs from the Kotlin field name
- `@Transient` (kotlinx, not the Java keyword) for excluded fields
- `Json { ignoreUnknownKeys = true }` on the shared `Json` instance
- `@Serializable` data classes don't have mutable `var` fields unless intentional and documented

## Date / time

- `kotlinx-datetime` (`Instant`, `LocalDate`, `Clock.System`) in common code — not `java.time`, `java.util.Date`, `NSDate`, or `Calendar`
- Time zones via `TimeZone.of(id)` — no hardcoded offsets

## Networking (Ktor)

- Ktor `HttpClient` configured in common code with platform engines injected via `expect`/`actual` or DI — not OkHttp or `URLSession` in common
- `HttpClient` created once and reused (or scoped to a component lifetime) — not per-request
- Timeouts configured (`HttpTimeout` plugin) — no unbounded requests
- Error responses handled via `response.status.isSuccess()` or `HttpResponseValidator` — not silent null returns

## Compose Multiplatform (CMP) — shared UI

Applies to shared Composables in `commonMain` that render on both Android and iOS (Compose Multiplatform 1.10+: unified `@Preview` across platforms, Navigation 3 supported on non-Android targets, Compose Hot Reload stable).

- Composables in `commonMain` follow the same source-set hygiene as non-UI code — no `android.*`, `UIKit`, or `Foundation` types, no Android-only Material components leaking into a shared Composable
- `@Composable expect fun ... / actual` used only for genuine platform-UI gaps (a native picker, a platform-specific gesture) — not as a workaround for a Compose API that already exists cross-platform; check whether the same effect is achievable with a plain shared Composable first
- Shared strings/images/fonts go through Compose Multiplatform resources (`commonMain/composeResources/`, the generated `Res` object) — `stringResource(Res.string.x)` / `painterResource(Res.drawable.x)`, not a hardcoded per-platform string or a raw resource path. A new user-facing string gets a `values/strings.xml` entry; a translation gets its own `values-<lang>/`
- Resources load synchronously on the caller's thread by default (raw/web resources are the exception) — a large image or font loaded straight from `painterResource` on a hot path can still janky-block composition; decode/cache off that path if it's large
- Navigation goes through `navigation-compose` (or Navigation 3, now cross-platform) driving one shared nav graph — not a hand-rolled per-platform router unless there's a genuine platform-only requirement
- ViewModel scoping uses the KMP `lifecycle-viewmodel-compose` (or `lifecycle-viewmodel-navigation3` under Nav3) artifact in `commonMain` — never `androidx.lifecycle.ViewModel` imported directly into common code, and never a plain class reconstructed on every recomposition in its place
- `ComposeUIViewController { }` configuration on the iOS side is set deliberately per screen — safe-area/insets aren't inherited automatically the way SwiftUI inherits them, and options like `enableBackGesture` default to a value that may not match the screen's actual navigation model
- Back-gesture/back-stack interop between the shared Compose nav graph and a surrounding UIKit `UINavigationController` reviewed explicitly when both exist — two independent back stacks listening for the same swipe gesture is a real bug, not a theoretical one
- CMP UI tests (Compose UI testing APIs run against `commonTest`/shared test source sets) cover shared Composable state and layout logic — not silently skipped on the iOS target because the test harness there is newer/less familiar

## Dependency injection (KMP)

- DI modules (Koin or the project's chosen framework) defined in `commonMain`; only the platform-actual bindings (a platform HTTP engine, a platform storage implementation) live in `androidMain`/`iosMain` — not a parallel service locator duplicated per platform
- Constructor injection into shared business logic, not `getKoin()`/service-locator calls scattered through `commonMain` — the latter makes shared code harder to unit test without a DI container running

## iOS / Swift interop (engine-ios-bindings and iosMain)

- Public Kotlin declarations for Swift consumption annotated with `@ObjCName("SwiftFriendlyName")` where the default name would be awkward in Swift
- Kotlin exceptions crossing the Swift boundary are caught and converted to result types, or declared with `@Throws(...)` — uncaught Kotlin exceptions terminate the iOS app
- `Flow` not directly exposed to Swift — wrapped via SKIE, KMP-NativeCoroutines, or a `CFlow`/callback helper
- `suspend` functions exposed to Swift use SKIE/KMP-NativeCoroutines or a callback wrapper — raw `suspend` is not ergonomically callable from Swift
- No Kotlin `object` singletons holding mutable state shared across threads without concurrency protection

## KMP testing

- New common logic tested in `commonTest` — not only in `androidTest`/JVM (which validates one target)
- `kotlinx.coroutines.test.runTest` — not `runBlocking` (JVM-only, unavailable in `commonTest`)
- Platform `actual`s tested in their platform test source set where behavior differs
- No `java.io.*` or `androidx.test.*` in `commonTest` — breaks the iOS build
- `kotlin.test.*` annotations (`@Test`, `@BeforeTest`, `@AfterTest`) — not JUnit annotations in common code

## KMP build / Gradle hygiene

- New KMP dependencies scoped to the right source set (`commonMain.dependencies { }`, `androidMain…`) — not plain `implementation` which targets only one platform
- `iosSimulatorArm64` included alongside `iosArm64` (and `iosX64` if still supported) — omitting it breaks M-series simulators
- XCFramework / `podspec` / SPM binary target updated if the public API surface changed and iOS consumers need to re-integrate
- Artifact version bumped (`publishToMavenLocal` / registry) if this is a library module with external consumers
