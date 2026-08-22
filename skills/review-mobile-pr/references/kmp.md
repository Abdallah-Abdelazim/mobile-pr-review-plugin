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
