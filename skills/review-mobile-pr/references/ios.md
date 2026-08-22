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
9. [Testing](#testing)
10. [Swift quality](#swift-quality)
11. [Localisation & accessibility](#localisation--accessibility)
12. [Project / build hygiene](#project--build-hygiene)

---

## Deprecations & platform changes (2026)

Context: since **April 28, 2026**, every App Store upload must be built with the **iOS 26 SDK / Xcode 26** or later — no exceptions. Privacy Manifests (`PrivacyInfo.xcprivacy`) are mandatory, and "required reason" APIs need declared reasons. Swift 6 strict concurrency is the compiler default for new modules.

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
