# Engineering excellence & code smells

Applies to **every PR**, regardless of platform. Two parts: the code-smell scan (fast, mechanical) and the excellence standards (judgment-based). Only flag `+` lines.

---

## Part 1 — Code smell scan

### Unused stuff

- **Unused imports** added in this diff (or left behind after this diff removes the last usage)
- **Unused parameters** on new/changed function signatures — if a param is intentionally unused (interface conformance), it should be named `_` (Swift) or annotated/suppressed with rationale (Kotlin)
- **Unused local variables** and **assigned-but-never-read** values
- **Ignored return values** where the return carries meaning (`Result`, a new object, a boolean success flag) — Kotlin: a dropped `Result`; Swift: an un-annotated discard of a `@discardableResult`-less return
- **Private functions/properties never referenced** — introduced dead weight
- **Feature flags / config keys added but never read**, or read but the old flag never removed when this PR completes the rollout
- **Resources added but unreferenced** (strings, drawables, assets) — and resources orphaned by deletions in this PR

### Dead code

- **Commented-out code blocks** — version control remembers; the file shouldn't
- **Unreachable code**: after `return`/`throw`, behind always-false conditions, behind a flag hardcoded off
- **`TODO`/`FIXME` that dodges this PR's actual work** — a TODO for a genuinely separate follow-up with a ticket reference is fine; a TODO in place of the error handling this change requires is not
- **Debug leftovers**: `print`/`println`/`Log.d` used as scaffolding, temporary hardcoded values, disabled tests (`@Ignore`, `.disabled`) without a linked reason

### Duplication

- Copy-pasted logic **within the diff** that should be one function
- New code duplicating an **existing utility** in the codebase — grep for a distinctive fragment before assuming it's novel
- Near-identical branches differing by one value — parameterize
- Test boilerplate repeated where a helper/parameterized test would serve

### Poorly written code

- **Functions doing several things** — mixed abstraction levels, "and" in the natural description of what it does; suggest extraction with names
- **Deep nesting** (> ~3 levels) — invert conditions, early return/`guard`
- **Magic numbers/strings** — named constants with intent (`MAX_RETRIES = 3`, not a bare `3`); exempt obvious zero/one/empty
- **Misleading names**: `getX()` that mutates, `isEnabled` that isn't a Bool, a `Manager`/`Helper`/`Util` grab-bag absorbing unrelated logic
- **Boolean parameters that obscure call sites** — `render(true, false)` tells the reader nothing; enums or separate functions
- **Long parameter lists** (> ~5) — group into a type
- **Primitive obsession** on domain concepts crossing module boundaries (raw `String` for an ID/URL/phone that has a type or should)
- **Inconsistency with the surrounding file** — new code that ignores the established pattern of the file it's in (different DI style, different error pattern) without stated reason
- **Redundant/duplicated state** — two properties/state vars added by this diff that must always agree (a list plus a separately-tracked `isEmpty`/`count`, a raw value plus a cached formatted copy) instead of one held value and the other derived from it on read
- **Efficiency smells, scoped to what this diff changed** — repeated work the diff could cache/hoist/share instead (recomputing the same value, re-parsing, re-fetching data already in scope); independent operations run sequentially with no data dependency between them where the diff could run them concurrently at negligible risk (parallel `launch`/`async` in a `coroutineScope`, `async let`/`withTaskGroup` in Swift, instead of back-to-back awaits). Flag only where the diff itself introduces or extends the shape — not a pre-existing pattern it merely touches in passing.

---

## Part 2 — Software-engineering excellence standards

### Design principles (applied pragmatically — flag violations that will cost, not theory)

- **Single responsibility**: a class/file modified for two unrelated reasons in this same PR is a hint it has two jobs
- **Open for extension**: a `when`/`switch` on a type code that this PR extends for the Nth time — the shape wants polymorphism or a sealed hierarchy with behavior
- **Dependency direction**: domain/business logic doesn't import UI or framework types; new dependencies point inward
- **Interfaces honest**: an implementation that throws "not supported" for part of its protocol/interface signals a wrong abstraction
- **Composition over inheritance** for new hierarchies; a new subclass overriding half its parent to negate behavior is a smell
- **YAGNI**: speculative abstraction (a protocol/interface with one impl and no test seam need, a config option nothing sets) adds cost now for value never
- **Leaky abstractions**: an internal implementation detail — a Room/Core Data entity, a raw network DTO, a platform-specific type — escaping through a public/domain-facing API instead of being mapped at the boundary, forcing callers to know about a layer they shouldn't

### Error handling standards

*Backstop only — the Bug Hunter pass's dedicated error-handling lens owns this territory with far more platform-specific depth; only act here on something it would clearly miss.*

- Errors handled at the level that can act on them — not caught-and-logged at every layer
- No empty catch blocks; no `catch` that converts a specific failure into a silent default without justification
- User-facing failures produce a user-facing state (error UI, retry), not just a log line
- Failure messages carry context (what operation, what input class — never PII)

### Naming & readability

- Names reveal intent at the call site; the reader shouldn't need the implementation
- Consistent vocabulary: one concept, one name across the diff (don't mix `fetch`/`load`/`get` for the same operation)
- Positive boolean names (`isEnabled` not `isNotDisabled`)
- Public API names follow platform conventions (Kotlin/Swift API design guidelines)

### PR scope & hygiene

- **One concern per PR**: a feature + an unrelated refactor + a formatting sweep buried together makes review and revert impossible — suggest splitting when the mix hides risk (note it once on the most affected file, not per-file)
- **Diff noise**: mass reformatting/import-reordering of untouched lines obscures the real change
- **PR description** states what and why; risky changes note the rollout/rollback plan; breaking changes called out
- **Migration completeness**: if the PR introduces a new pattern to replace an old one, either the old one is fully migrated or the follow-up is ticketed — a codebase with three ways to do one thing is worse than either way alone
- **Generated files** not hand-edited; lockfiles/`libs.versions.toml`/`Package.resolved` changes match the stated dependency change

### Test quality standards

*Backstop only — the Test Analyzer pass owns this territory; only act here on a structural issue it wouldn't catch (e.g. wrong source-set placement).*

- New behavior has tests; changed behavior has **changed** tests (a behavior change with zero test diff means the behavior wasn't covered — flag it)
- Tests assert outcomes, not implementation details (over-mocked tests that verify call sequences break on every refactor)
- Test names state the scenario and expectation
- No assertion-free tests, no tests that can't fail
- Flaky patterns: real clocks, real network, sleeps, order-dependent tests

### Documentation & comments

- Comments explain **why**, not what — a comment paraphrasing the line below is noise; a comment explaining a non-obvious constraint or workaround (with link) is gold
- Public API surface (new public functions/types consumed outside the module) carries doc comments
- Complex algorithms/regexes/bit-twiddling get an intent comment
- Stale comments contradicted by the code change in this PR are updated, not left lying
