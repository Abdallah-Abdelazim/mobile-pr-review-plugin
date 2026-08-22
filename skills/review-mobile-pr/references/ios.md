# iOS review reference (2026)

Apply to Swift/SwiftUI/UIKit/Xcode files. Only review lines present in the diff (`+` lines). Never flag pre-existing code.

## Table of contents

1. [Deprecations & platform changes (2026)](#deprecations--platform-changes-2026)
2. [Swift 6 concurrency](#swift-6-concurrency)
3. [SwiftUI](#swiftui)
4. [UIKit](#uikit)
5. [Memory management](#memory-management)
6. [Error handling & optionals](#error-handling--optionals)
7. [Security & privacy](#security--privacy)
8. [Performance](#performance)
9. [SwiftData](#swiftdata)
10. [WidgetKit, Live Activities & App Intents](#widgetkit-live-activities--app-intents)
11. [StoreKit 2](#storekit-2)
12. [Background tasks (BGTaskScheduler)](#background-tasks-bgtaskscheduler)
13. [Push & local notifications](#push--local-notifications)
14. [Swift macros](#swift-macros)
15. [Testing](#testing)
16. [Swift quality](#swift-quality)
17. [Localisation & accessibility](#localisation--accessibility)
18. [Project / build hygiene](#project--build-hygiene)

---

## Deprecations & platform changes (2026)

Context: since **April 28, 2026**, every App Store upload must be built with the **iOS 26 SDK / Xcode 26** or later — no exceptions. Privacy Manifests (`PrivacyInfo.xcprivacy`) are mandatory, and "required reason" APIs need declared reasons. Swift 6 strict concurrency is the compiler default for new modules. iOS 26 unified Apple's OS versioning (iOS/iPadOS/macOS/watchOS/tvOS/visionOS all on the "26" cycle) — treat any codepath still branching on `#available` for the old numbering scheme (iOS 17/18) as still valid, but new `#available(iOS 26, *)` checks are the current baseline.

| Newly added usage of… | Status | Replacement / note | Severity |
|---|---|---|---|
| `UIWebView` | Removed; apps rejected | `WKWebView` | 🔴 |
| `ObservableObject` + `@Published` + `@StateObject`/`@ObservedObject`/`@EnvironmentObject` in **new** view models (iOS 17+ deployment target) | Superseded by the Observation framework | `@Observable` macro + `@State` (ownership) / plain property (passed) / `@Bindable` (two-way) / `@Environment` — gives property-level tracking and fewer re-renders. **Migration trap:** `@State` initializes its value on every view rebuild, unlike `@StateObject`'s lazy `@autoclosure` — flag expensive VM construction in the view initializer. | 🟡 |
| `NavigationView` | Deprecated | `NavigationStack` / `NavigationSplitView` with typed `NavigationPath` | 🟡 |
| `PreviewProvider` boilerplate in new previews | Superseded | `#Preview` macro | 🟢 |
| Old alert/sheet APIs (`Alert(title:…)`, `.alert(isPresented:content:)` returning `Alert`) | Deprecated | `.alert(_:isPresented:actions:message:)` builder APIs | 🟢 |
| `foregroundColor(_:)` | Deprecated | `foregroundStyle(_:)` | 🟢 |
| `.animation(_:)` (no value) | Deprecated (ambient animation) | `.animation(_:value:)` or `withAnimation { }` | 🟡 |
| `onChange(of:) { newValue in }` single-param | Deprecated iOS 17 | Two-parameter `onChange(of:) { old, new in }` or zero-param | 🟢 |
| `DispatchQueue`/GCD in new async code paths | Superseded | Swift Concurrency: `Task`, `async/await`, actors, `AsyncSequence`; `@MainActor` instead of `DispatchQueue.main.async` for UI hops | 🟡 |
| New Combine pipelines for one-shot async work | Not deprecated, but Apple investment is in Swift Concurrency | Prefer `async/await` / `AsyncSequence` for new code; Combine fine where the codebase is already Combine-based | 🟢 |
| Completion-handler APIs where an async overload exists (`URLSession.dataTask` vs `data(for:)`) | Superseded | Async overloads | 🟡 |
| Core Data for brand-new persistence in a greenfield module (iOS 17+) | Superseded for new work | SwiftData (`@Model`) — but do **not** flag additions to an existing Core Data stack | 🟢 |
| `NSCoding`/`NSKeyedArchiver` without secure coding | Insecure/deprecated pattern | `Codable`, or `NSSecureCoding` with `requiresSecureCoding = true` | 🟠 |
| XCTest for brand-new unit-test targets | Superseded for new pure-Swift tests | Swift Testing (`@Test`, `#expect`, `#require`) — don't flag additions to existing XCTest suites; UI tests remain XCTest | 🟢 |
| Storyboard/XIB additions for new screens in a SwiftUI-first codebase | Legacy direction | SwiftUI (or the project's established UIKit pattern) | 🟢 |
| Required-reason APIs (`UserDefaults`, file timestamps, disk space, boot time…) added without a `PrivacyInfo.xcprivacy` entry | Store rejection risk | Declare the reason in the privacy manifest | 🟠 |
| `@preconcurrency import` added to silence warnings | Escape hatch | Acceptable at true legacy boundaries; flag when used to dodge fixing the module's own isolation | 🟡 |
| New user-facing feature with a custom in-app UI screen and no `AppIntent` equivalent | Discovery gap, not a hard deprecation | Expose the action via `AppIntent` so it surfaces in Siri/Spotlight/Shortcuts/widgets — App Intents are the default discovery surface as of iOS 26 | 🟡 |
| `NavigationLink(destination:isActive:)` / other Boolean-driven navigation in new code | Superseded | Value-driven `NavigationStack(path:)` / `NavigationLink(value:)` | 🟢 |

If the diff uses an API you don't recognize or you're unsure whether it's been deprecated since this file was written, search the web before commenting.

---

## Swift 6 concurrency

The compiler catches many data races, but reviews still catch design errors the compiler permits:

- **Actor isolation is deliberate**: `@MainActor` on UI-facing types/functions; heavy work (`decode`, image processing, disk I/O) **not** trapped on the main actor — move to a background task or `@concurrent`/nonisolated function
- No blocking calls (`sleep`, sync I/O, semaphores, `DispatchSemaphore.wait`) inside async contexts — starves the cooperative thread pool
- `Task { }` created from a view/controller is cancelled when its owner goes away (`.task { }` modifier auto-cancels; raw `Task` stored and cancelled in `deinit`/`onDisappear`)
- Detached tasks (`Task.detached`) justified — they drop actor context and priority; almost always the wrong default
- `Sendable` conformances are real: no `@unchecked Sendable` on types with mutable state unless protected by a lock/queue and documented
- Long-running loops check `Task.isCancelled` / call `Task.checkCancellation()`
- `withTaskGroup` for a dynamic number of parallel tasks; `async let` for a fixed small set
- Shared mutable state lives in an `actor` (or is immutable) — not a class with ad-hoc locking
- No fire-and-forget `Task` that swallows thrown errors — handle or log
- Race between `Task` start and view state: don't read `self`-mutable state after an `await` without re-validating it
- **`sending` used instead of blanket `Sendable`** where a non-`Sendable` value only needs to cross an isolation boundary once and isn't touched again by the caller afterward — cheaper and more precise than making the whole type conform to `Sendable`
- **Region-based isolation relied on for local data flow** (a value created and consumed entirely within one isolation domain) rather than reaching for `@unchecked Sendable` as a shortcut — if a type needs `@unchecked Sendable` to compile, that's usually a sign the data actually crosses isolation domains and needs real protection
- **Strict-concurrency checking level is consistent across the target**: a module still compiling at `Minimal`/`Targeted` while the rest of the app is Swift 6 language mode gives a false sense of safety at the boundary — flag a new module or package added without matching the app's concurrency mode

## SwiftUI

- State ownership correct: `@State` for view-owned values and `@Observable` models the view creates; plain `let`/`var` for models passed in; `@Bindable` only when a two-way binding is needed
- View `body` is pure — no side effects, no object creation with side effects, no network calls; use `.task`/`.onAppear`
- Expensive computation not in `body` — precompute in the model or memoize
- `ForEach` uses stable, unique `id`s — never `id: \.self` on non-unique data, never array indices for mutable lists
- View decomposition: giant `body` blocks split into subviews or computed properties; `@ViewBuilder` used where appropriate
- Navigation: typed `NavigationPath`/destination values; navigation state owned by the model, not scattered `isActive` booleans
- `.task(id:)` used when the async work must restart on an input change
- No `GeometryReader` wrapping whole screens when alignment/layout primitives suffice (layout thrash)
- Animations attached to specific value changes (`.animation(_:value:)`), not ambient
- Environment values over deep parameter drilling for cross-cutting concerns (theme, locale)
- **Views conform to `Equatable`** (or wrap an expensive subtree in `EquatableView`) when `body` computation is nontrivial and its inputs rarely change — without it, SwiftUI re-diffs the subtree on every parent update regardless of whether anything relevant changed
- **`@Bindable` scoped to the object whose properties are actually bound** to a child control — applying `@Bindable` to a passed-in model "just in case" opts the whole subtree into observation it doesn't need, and can widen invalidation beyond what actually changed

## UIKit

- View controller lifecycle respected: subscriptions started in `viewWillAppear` are torn down in `viewDidDisappear`
- No work in `viewDidLoad` that depends on final layout — use `viewDidLayoutSubviews` or constraints
- Delegates are `weak`
- `UITableView`/`UICollectionView`: cell reuse correct (no stale async images — cancel/validate on reuse); diffable data sources preferred over `reloadData`
- Main-thread-only UIKit APIs never touched from background contexts (`UIView`, `UIViewController`, `UIApplication`)
- Auto Layout constraints deactivated/updated, not stacked duplicates on every pass

## Memory management

- **Retain cycles**: escaping closures stored by `self` capture `[weak self]` (or `[unowned self]` only when lifetime is provably bound); `guard let self else { return }` after weak capture
- Timers, `NotificationCenter` observers (block-based API returns a token — store and remove it), KVO observations invalidated/removed
- Combine `AnyCancellable`s stored (`store(in: &cancellables)`) and cancelled with their owner
- No accidental strong reference from a long-lived object (singleton, static) to a short-lived one
- Large data (images, buffers) not held in caches without eviction (`NSCache` over dictionaries)

## Error handling & optionals

- No new force unwraps (`!`), `try!`, or `as!` outside of tests/previews — `guard let`, `try?` with handling, or thrown errors with context
- `try?` doesn't silently discard errors on critical paths — at minimum log; ideally propagate
- Errors are typed/domain-specific where callers branch on them; raw `NSError` codes not stringly matched
- `guard` used for early exit; nesting kept shallow
- Optionals model true absence — not used as a lazy substitute for proper initialization

## Security & privacy

- No secrets/API keys in source, plists, or `xcconfig` committed to the repo
- Keychain (not `UserDefaults`) for tokens and credentials
- ATS not weakened (`NSAllowsArbitraryLoads` never introduced); pinned certs changed deliberately
- User data not logged (`print`, `os_log`) — use privacy-redacting `Logger` (`\(value, privacy: .private)`)
- New data collection or required-reason API usage reflected in `PrivacyInfo.xcprivacy`
- Deep links / universal links validate parameters before acting
- Pasteboard, contacts, photos, location access gated behind purpose strings that match actual usage

## Performance

- Image decoding/downsampling off the main thread and sized to the display target
- No synchronous disk/network on the main actor
- `LazyVStack`/`LazyHStack` (or list virtualization) for long scrolling content
- Repeated `DateFormatter`/`NumberFormatter`/`JSONDecoder` creation hoisted — they're expensive
- String concatenation in loops replaced with efficient building where it matters

## SwiftData

- Any breaking `@Model` change (property added/removed/retyped, relationship reshaped) ships a `SchemaMigrationPlan` stage — not left to lightweight-migration inference when the change isn't actually lightweight-compatible (renamed properties, split/merged properties, and type changes generally aren't)
- Non-trivial migrations use `MigrationStage.custom` with `willMigrate`/`didMigrate` closures to move/backfill data — a `.lightweight` stage silently drops data it can't infer a mapping for
- `@Attribute(.unique)` declared on any field the domain actually requires unique — its absence lets duplicate rows accumulate silently
- Relationships declare `deleteRule` explicitly (`.cascade` / `.nullify` / `.deny`) — the implicit default (`.nullify`) is rarely correct for an owns-its-children relationship, and an unreviewed default can orphan or silently null out data
- Multi-step or multi-object writes that must be atomic happen inside a single `save()` — not several partial saves that can leave inconsistent state if the app is killed between them
- Background writes use their own `ModelContext`/`ModelActor` — the main-actor `modelContext` from the environment is never passed across threads and mutated concurrently

## WidgetKit, Live Activities & App Intents

- A new user-facing action exposed only through a custom in-app screen, with no `AppIntent` equivalent — App Intents are the default discovery surface (Siri, Spotlight, Shortcuts, widgets) as of iOS 26; flag the missing intent as a discoverability gap, not just style
- Interactive widget controls use `Button(intent:)` / `Toggle(isOn:intent:)` backed by an `AppIntent` whose `perform()` runs inside the widget extension — no attempt to launch the host app or read app-process-only state that isn't available to the extension
- `AppIntent.perform()` bodies stay fast and self-contained, with bounded work — the extension runs under a tight time budget and unbounded network/disk calls risk the system killing it mid-execution
- Live Activity push-to-start payloads and periodic updates stay under the system's payload size budget — an oversized payload fails **silently** (telemetry only, no crash or error), so "the Live Activity never updates" should be triaged as a payload-size bug first
- `ActivityAuthorizationInfo().areActivitiesEnabled` checked before starting an `Activity` — never assumed to be permitted
- `TimelineProvider`/`AppIntentTimelineProvider` entries computed cheaply; no blocking network call in `snapshot()`/`timeline()` without a placeholder fallback for the case it can't complete in time

## StoreKit 2

- Every transaction verified via `VerificationResult`/`checkVerified` **before** entitlement is granted — an unverified transaction is never trusted
- `Transaction.updates` (and `currentEntitlements` at launch) observed by a long-running `Task` started at app launch — checking only in response to a user-initiated purchase misses renewals, refunds, and Family Sharing changes made outside the app
- Every transaction calls `transaction.finish()` once its entitlement is granted — an unfinished transaction replays on every subsequent launch
- Refunds and revocations handled (`Transaction.revocationDate`/`revocationReason`) by removing the entitlement, not just granting it once and forgetting
- Subscription state read from `Product.SubscriptionInfo.status`, not inferred solely from the existence of a past transaction, which doesn't reflect current renewal/grace-period/billing-retry state

## Background tasks (BGTaskScheduler)

- `BGTaskScheduler.shared.register(...)` called unconditionally during app launch, before `application(_:didFinishLaunchingWithOptions:)` returns — a registration added later, or gated behind a condition, silently no-ops for that task identifier
- Submitted requests declare constraints (`requiresNetworkConnectivity`, `requiresExternalPower`) and a realistic `earliestBeginDate` that actually match what the task needs — an always-immediate, no-constraint request gets deprioritized by the system scheduler
- The task's `expirationHandler` cancels in-flight work and calls `task.setTaskCompleted(success:)` — a task that never signals completion stops the system from scheduling that identifier again
- Long-running background work is chunked so it can checkpoint progress — background execution time is never guaranteed to run to completion

## Push & local notifications

- Authorization requested with only the options actually used (`.alert`, `.sound`, `.badge`, `.provisional`) — not a blanket request with no context for why
- `UNUserNotificationCenterDelegate` assigned before the first notification could plausibly arrive (typically in `application(_:didFinishLaunchingWithOptions:)`) — assigning it later misses early notifications
- Notification content mutated via `UNNotificationServiceExtension` only when the payload actually sets `mutable-content`
- Deep-link / user-info payloads carried on a notification validated before driving navigation — same rule as any other deep link; a malformed or spoofed payload shouldn't drive privileged navigation
- Category/action identifiers handled exhaustively in the delegate's `didReceive` — a new action identifier added later should hit a defined fallback, not silently do nothing

## Swift macros

- Custom macro expansions follow this file's other rules just like hand-written code — a macro that generates a force-unwrap, a retain cycle, or a `Sendable`-unsafe capture is worse than the equivalent hand-written bug because it's invisible at the call site
- A PR adding or changing a macro's expansion reviewed via its expanded output (Xcode's "Expand Macro," or the compiler's generated-code diagnostic), not approved on the macro definition's intent alone
- Property-wrapper-style macros (`@Model`, `@Observable`, `@AppStorage`) not stacked in combinations the macro isn't documented to support — undefined interactions between macros are a common source of silent data loss or lost observation

## Testing

- New logic has unit tests; async code tested with `async` test functions — no `XCTestExpectation` gymnastics where `await` suffices, no sleeps
- Swift Testing (`@Test`, `#expect`, `#require`) for new pure-Swift test targets; parameterized tests over copy-pasted cases
- `@MainActor` on tests exercising main-actor-isolated types
- Test doubles injected via protocols/initializers — no live network in unit tests
- Tests actually exercise the code under test: verify the test calls the function/fires the event it claims to test
- Snapshot/screenshot tests follow the project's established base classes and helpers — scan existing tests before approving new structure

## Swift quality

- Value types (`struct`/`enum`) preferred where identity isn't needed; classes justified
- `enum` with associated values over parallel optionals for mutually exclusive states
- `switch` over enums exhaustive — no `default` that swallows future cases (use `@unknown default` only for non-frozen system enums)
- Access control tightest that works: `private`/`fileprivate` by default; `public` API changes are deliberate
- Protocol-oriented where abstraction is needed — but no single-conformer protocols invented purely for a mock when a simpler seam exists
- No stringly-typed identifiers where an enum/constant works
- `defer` for cleanup paired with acquisition
- Fully qualified names replaced with imports; unused imports removed

## Localisation & accessibility

- User-visible strings in string catalogs (`.xcstrings`) / `NSLocalizedString` — no hardcoded literals
- Pluralization via string catalog plural rules, not `count == 1` branching
- Format specifiers positional when multiple (`%1$@`), escaping correct
- Dynamic Type respected: no fixed font sizes on user text; layouts survive larger sizes
- VoiceOver: interactive elements have labels; decorative images hidden (`accessibilityHidden(true)`); traits (`.isButton`, `.isHeader`) correct
- Touch targets ≥ 44×44pt

## Project / build hygiene

- New dependencies via SPM with pinned versions (no branch-based deps in production); rationale for each new dep
- No `.xcodeproj` merge damage: duplicated build-file entries, orphaned references
- Build settings changed in `.xcconfig`/project deliberately — never silence warnings globally to land a PR (`@Diagnose`/targeted suppression over blanket flags)
- New targets/schemes wired into CI
- `Info.plist` additions (URL schemes, background modes, purpose strings) match actual features
