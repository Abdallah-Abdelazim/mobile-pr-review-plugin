---
name: review-mobile-pr
model: claude-sonnet-5
description: Expert Android & iOS PR review. Defaults to saving findings as a PENDING (draft) GitHub review — invisible until manually submitted — but will post them live instead if the user asks or says so when prompted. Use whenever the user asks to review, audit, or give feedback on a pull request touching Android (Kotlin, Jetpack Compose, Gradle), iOS (Swift, SwiftUI, UIKit), or KMP code — including phrases like "review this PR", "check my PR", "draft review", or a GitHub PR URL for a mobile repo. Reviews against up-to-date (2026) platform deprecations, Swift 6 / Compose best practices, code smells (unused code, dead code, poor structure), and software-engineering excellence standards.
---

# Mobile PR Review — Expert Android & iOS Engineer

Reviews a GitHub PR through the lens of a **senior mobile engineer** (Android, iOS, and KMP) and, by default, saves all findings as a **pending (draft) review** — comments are visible only to you in the GitHub UI until you choose to submit them. The user can ask for findings to go live immediately instead; see "Posting mode" below.

This skill ships in the `mobile-pr-review` plugin with its own dedicated review agents (`agents/mobile-pr-*.md`) — it does not depend on any other plugin. Dispatch each by its fully-qualified `mobile-pr-review:mobile-pr-*` name. The review runs as a set of specialized agents dispatched in parallel, each a self-contained mobile-review specialist:

| Agent | Focus | Dispatch |
|---|---|---|
| `mobile-pr-review:mobile-pr-bug-hunter` | Correctness — forgotten call sites, unhappy paths, wrong logic, non-exhaustive branching, contract mismatches, concurrency correctness | Always |
| `mobile-pr-review:mobile-pr-silent-failure-hunter` | Swallowed exceptions, unjustified fallbacks, overly broad catches | Always |
| `mobile-pr-review:mobile-pr-code-quality-reviewer` | Code smells, dead/unused code, duplication, SOLID/naming/PR-scope, platform checklist backstop | Always |
| `mobile-pr-review:mobile-pr-deprecation-scanner` | APIs deprecated/superseded/removed as of 2026 (Android 16/API 36, Swift 6, iOS 17–26) — extends beyond what any generic review plugin tracks | Always, when the diff touches Android and/or iOS files |
| `mobile-pr-review:mobile-pr-test-analyzer` | Test coverage gaps, tests that don't exercise what they claim to | Always |
| `mobile-pr-review:mobile-pr-comment-analyzer` | Comment/doc accuracy, stranded artifacts from incomplete deletions | When the diff adds/modifies comments or doc comments |
| `mobile-pr-review:mobile-pr-type-design-analyzer` | Type encapsulation and invariant expression | When the diff adds/reshapes a data class, sealed class/interface, enum, struct, or protocol |

See step 3 for how each is dispatched and step 4 for the deprecation pass specifically.

## 📮 Posting mode

Before doing anything else, work out how the findings should be posted:

- **Draft (the default)** — saved as a pending review, invisible to everyone but the author of the review until they open the PR and submit it themselves.
- **Live** — posted the moment this skill finishes, visible to everyone on the PR immediately.

Resolve this, in order:

1. **An explicit flag in the invocation** (see Usage below: `--live`/`--now` selects Live, `--draft` selects Draft) — skip the question if one is present.
2. **Otherwise, ask once, before pre-flight:** *"Should I hold these findings as a private draft you review and submit yourself, or post them live the moment I'm done? Draft is the default — just say so if you'd rather they go live right away."* Treat silence, "either," "you choose," or any other non-committal answer as **Draft**.

Carry the resolved mode through the rest of the workflow — it decides the `event` field in step 7, the header in step 1, and the wording of the summary in step 8.

## ⛔ Safety contract (read this first)

- **Draft is the safe default and stays invisible until the human submits it.** Only switch to Live when the posting mode above actually resolved to Live — never post live "just in case" or because it seemed faster.
- **Draft mode:** never use `event: APPROVE`, `event: REQUEST_CHANGES`, or `event: COMMENT` — all three submit immediately. Omit the `event` field entirely; that's what keeps the review pending.
- **Live mode:** use `event: COMMENT` only. Never `APPROVE` or `REQUEST_CHANGES` — this skill reports findings, it doesn't approve or block a PR, regardless of posting mode.
- **Never use `gh pr review --comment`, `gh pr comment`, or the issues comments API**, in either mode — always the single `pulls/<number>/reviews` call in step 7, so every comment lands together in one review.
- **Leave `body` empty (`""`)** in both modes — a non-empty body becomes a visible summary comment the moment the review is submitted (Draft) or posted (Live), and may not reflect the final, cross-checked set of findings.
- Comment on diff `+` lines, or on a context/`-`-adjacent line that this diff left stale or orphaned (see "stranded artifacts from incomplete deletions" — caught by `mobile-pr-review:mobile-pr-comment-analyzer` and `mobile-pr-review:mobile-pr-code-quality-reviewer`) — but don't flag pre-existing code the diff never touched.
- Use the authenticated GitHub account shown in `gh auth status`.
- **If the reviews API call fails, do not fall back to any other posting mechanism, in either mode.** Report the error in the terminal and tell the user to post manually. A failed post is better than an accidental or malformed one.

## Usage

Invoke with a PR URL or number, optionally naming a posting mode to skip the prompt:
```
/review-mobile-pr https://github.com/<org>/<repo>/pull/<number>
/review-mobile-pr <number>          # when already inside the repo; asks Draft-or-Live before posting
/review-mobile-pr <number> --live   # skip the question — post live immediately
/review-mobile-pr <number> --draft  # skip the question — save as a pending review (the default anyway)
```

## Reference files (read the relevant ones before reviewing)

| File | When to read it |
|---|---|
| `references/android.md` | Any Android/Kotlin/Compose/Gradle file in the diff. Full checklist: architecture, Compose, coroutines, lifecycle, DI, security, performance, testing, Kotlin quality, resources, accessibility, build hygiene — **plus the Android 2026 deprecation table.** |
| `references/ios.md` | Any Swift/SwiftUI/UIKit/Xcode file in the diff. Full checklist: Swift 6 concurrency, SwiftUI state, UIKit lifecycle, memory management, security, testing, Swift quality — **plus the iOS 2026 deprecation table.** |
| `references/kmp.md` | Any file under `kmp/` or in shared/multiplatform source sets. Source-set hygiene, expect/actual, KMP-safe concurrency, serialization, Ktor, Swift interop, KMP testing and Gradle rules. |
| `references/engineering-excellence.md` | Every PR, regardless of platform. Code smells, dead/unused code, SOLID, naming, error handling, PR scope & hygiene, documentation, test quality standards. |

These files live at `${CLAUDE_PLUGIN_ROOT}/skills/review-mobile-pr/references/` — e.g. `${CLAUDE_PLUGIN_ROOT}/skills/review-mobile-pr/references/android.md`. Read only the files matching the platforms actually present in the diff — plus `engineering-excellence.md`, which always applies regardless of platform. You don't need to paste their contents into agent prompts — pass each dispatched agent the **path(s)** to the reference files it needs (with `${CLAUDE_PLUGIN_ROOT}` expanded to this install's actual plugin root); every `mobile-pr-review:mobile-pr-*` agent has `Read` access and reads them itself.

## Workflow

### 1. Pre-flight

Resolve the posting mode first (see "Posting mode" above) if it isn't already clear from the invocation.

```bash
gh auth status   # must succeed — stop if not authenticated
```

Parse the PR URL/number to extract `owner`, `repo`, `pr_number`.

```bash
gh pr view <number> --repo <owner>/<repo> \
  --json number,title,body,baseRefName,headRefName,state,files
gh pr diff <number> --repo <owner>/<repo>
gh api repos/<owner>/<repo>/pulls/<number>/comments --paginate   # existing inline review comments
gh api repos/<owner>/<repo>/issues/<number>/comments --paginate  # existing top-level PR comments
```

Keep the existing comments on hand — you'll cross-check your findings against them before posting (step 6).

Show header (with the resolved posting mode):
```
🔍 Mobile PR Review (draft | live)
📋 PR #<number>: <title>
🔀 <base> ← <head>
📂 Files changed: <count>
```

### 2. Detect platform context

Map each changed path to a platform so the right reference file and checklist apply:

| Path / extension pattern | Platform | Reference |
|---|---|---|
| `*.kt`, `*.kts` under `android/`, `app/`, or Android modules | Android | `references/android.md` |
| `*.swift`, `*.xcodeproj`, `*.xcconfig`, `Podfile`, `Package.swift` | iOS | `references/ios.md` |
| `kmp/…/commonMain`, `commonTest`, `androidMain`, `iosMain`, `engine-ios-bindings` | KMP | `references/kmp.md` (plus android/ios refs for the respective actuals) |
| `*.gradle.kts`, `libs.versions.toml`, `gradle.properties` | Build (Android/KMP) | build-hygiene sections of `android.md` / `kmp.md` |
| CI workflows, scripts, docs | Cross-cutting | `engineering-excellence.md` only |

Read the matching reference files **now**, before starting the review passes. Skip categories with zero relevance to the file type.

### 3. Dispatch the review agents

Delegate the labor-intensive analysis to this skill's own bundled agents instead of doing it by hand. Launch all applicable agents **in parallel** — a single message with multiple `Agent` tool calls, one per agent, using the `mobile-pr-review:mobile-pr-*` names as `subagent_type`. Each is a fresh agent with no context of this conversation, so build a self-contained prompt for every one containing:

- **PR intent** — one line stating what the change is supposed to do and its happy path, from the PR title/description/linked ticket. You cannot judge "wrong" or "forgotten" without knowing "intended," and every agent needs this framing.
- **The full PR diff** (from `gh pr diff` in pre-flight) and the changed-files list.
- **Absolute path(s) to the relevant reference file(s)** — the platform file(s) from step 2's platform detection (`android.md` / `ios.md` / `kmp.md`), plus `engineering-excellence.md` unconditionally for `mobile-pr-review:mobile-pr-code-quality-reviewer` (it always applies, independent of platform — see the reference-files table above). Each agent reads these itself via its `Read` tool, so pass paths, not pasted excerpts.
- **An output-format request**: *"Return findings as a plain list, one per line: `<file>:<line> — <severity: CRITICAL/HIGH/MEDIUM/LOW> — <short title> — <issue and why it matters> — <suggested fix>`."* This lets step 5 fold results mechanically into the Comment Format below without re-interpretation.

Always dispatch:

| Agent | Focus |
|---|---|
| `mobile-pr-review:mobile-pr-bug-hunter` | Bug hunt — forgotten call sites, unhappy paths, wrong/non-exhaustive logic, contract mismatches, resource-lifecycle leaks, concurrency correctness. The highest-value pass; give it the PR intent and full diff. |
| `mobile-pr-review:mobile-pr-silent-failure-hunter` | Swallowed exceptions, inadequate error handling, unjustified fallbacks, overly broad catches — a dedicated adversarial lens on top of the bug hunt. |
| `mobile-pr-review:mobile-pr-code-quality-reviewer` | Code smells & hygiene, dead code, duplication, SOLID/naming/PR-scope standards, plus the platform checklist backstop (architecture, Compose/SwiftUI, DI, security, performance, a11y, localisation, build hygiene). |
| `mobile-pr-review:mobile-pr-test-analyzer` | Behavioral test coverage gaps, untested edge cases, and tests that don't actually exercise what they claim to. |

Dispatch conditionally, only when relevant to this diff:

| Agent | Include when |
|---|---|
| `mobile-pr-review:mobile-pr-comment-analyzer` | The diff adds/modifies comments, KDoc, or doc comments — also catches "stranded artifacts from incomplete deletions" (a comment left behind by a deletion elsewhere in the hunk). |
| `mobile-pr-review:mobile-pr-type-design-analyzer` | The diff adds or reshapes a `data class`, `sealed class`/`interface`, `enum class`, or a Swift `struct`/`protocol`/`enum`. |

The deprecation pass is dispatched separately in step 4, since it needs the deprecation-table reference files specifically and nothing else.

**Verify before trusting.** An agent above works only from what its prompt gave it. If a returned finding depends on something outside that diff (another call site, a default value, what a function returns), re-verify with `grep`/`Read` before accepting it into your findings pool — a wrong finding wastes the author's time and burns review credibility.

### 4. Deprecation & modernity pass

Dispatch `mobile-pr-review:mobile-pr-deprecation-scanner` (in the same parallel batch as step 3, or right after — either is fine) whenever the diff touches Android and/or iOS files. Give it the diff, the changed-files list, and the absolute path(s) to `android.md` and/or `ios.md` — whichever platform(s) apply. It reads the deprecation tables itself and flags newly-added usage of anything deprecated, removed, or superseded as of 2026 (Android 16/API 36, Swift 6, iOS 17–26), web-searching anything it doesn't recognize rather than guessing. Skip this dispatch entirely for a pure-KMP-common diff with no `androidMain`/`iosMain` files touched.

### 5. Aggregate findings

Collect every dispatched agent's raw output — each already carries `<file>:<line> — <severity> — <title> — <issue> — <fix>` per the output-format request in step 3 — into one findings pool. Reshape each into the Comment Format below when you get to posting, and discard any positive observations or summary line an agent's report also included ("if I found nothing, I said so in one line" — drop those lines from the pool); only carry forward concrete, file/line-anchored findings.

### 6. Cross-check against existing PR comments

Before posting, compare every finding in the aggregated pool from step 5 against the comments fetched in pre-flight (both inline review comments and top-level PR comments) so you don't duplicate feedback that's already on the PR — from an earlier draft pass, another reviewer, or a bot.

For each finding, look for existing comments **on the same file and the same line or line range**, then judge on substance, not exact wording — a comment saying "this will NPE on empty list" and one saying "add a null check before iterating" about the same line are the same finding even though the wording differs:

- **Same finding, already said** — an existing comment already flags the same underlying problem (same root cause, same location), even with a different severity label or phrasing → **drop it, do not post**. Count it toward "duplicates skipped" in the summary.
- **Close but not the same** — an existing comment touches the same line/area but raises a different angle, misses something yours catches, or only partially overlaps → **still post your finding**, and append a short note referencing the existing comment so the author can reconcile both:
  ```
  **Related existing comment**: @<author> already flagged something adjacent here: "<short quote or paraphrase>" — <one clause on how yours differs or adds>.
  ```
- **No overlap** — post normally, no mention needed.

When in doubt whether two comments describe the same root cause, treat them as merely "close" (post + reference) rather than "same" (drop) — a false duplicate-skip silently loses a finding, while a false "close" match only costs the author one extra sentence of context.

### 7. Post findings

- **No top-level PR comments** (`gh pr review --comment`, `gh pr comment`, `gh api .../issues/.../comments`) — in either mode, these post immediately and bypass the one-shot review call below
- **No review body/summary** — leave the `body` field empty (`""`) in both modes
- **Inline comments only**, scoped to specific diff lines

Use the GitHub API — the payload is identical in both modes except for one field:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews \
  --method POST \
  --input - <<EOF
{
  "body": "",
  "comments": [
    {
      "path": "<relative file path>",
      "line": <line number in the file>,
      "side": "RIGHT",
      "body": "<comment body>"
    }
  ]
}
EOF
```

**Draft mode: send the payload exactly as above, with no `event` field.** Omitting `event` is what tells the GitHub API to save the review as pending — invisible until manually submitted. Passing `"event": "PENDING"` returns a 422 error.

**Live mode: add `"event": "COMMENT"` to the top-level object** (alongside `"body"` and `"comments"`). This posts the review — and every inline comment in it — the moment the call succeeds. Never pass `APPROVE` or `REQUEST_CHANGES` here; this skill reports findings, it doesn't gate the PR.

For a multi-line finding, add `"start_line": <first line>` and `"start_side": "RIGHT"` alongside `line` (the last line of the range) — in either mode.

### Comment format

Each inline comment body should follow:

```
<emoji> <SEVERITY>: <Short Title>

**Issue**: <what is wrong and why — platform-specific where relevant>

**Why it matters**: <crash, leak, recomposition storm, data race, silent data loss, store rejection, maintenance cost, etc.>

**Fix**:
```kotlin or ```swift
<corrected code>
```
```

**When the fix is a direct, single-line replacement** (missing default value, wrong import, unused line, trivial rename), use a GitHub suggestion block instead of a language code block. This lets the author apply the fix with one click:

````
```suggestion
    subtitle: String? = null,
```
````

Use `suggestion` when:
- The fix is a 1–3 line drop-in replacement for the highlighted line(s)
- No surrounding context needs to change
- The correct code is unambiguous

Use a language block (not suggestion) when:
- The fix spans multiple non-contiguous locations
- A design decision or explanation is more valuable than the exact code
- The replacement requires context the author must supply

Severity scale:
- 🔴 CRITICAL — crash, data loss, data race, security exploit, memory/context leak, guaranteed store rejection
- 🟠 HIGH — incorrect coroutine/task scope or actor isolation, lifecycle violation, deprecated API with a hard migration deadline, untested critical path
- 🟡 MEDIUM — recomposition/render inefficiency, missing error handling, superseded API, DRY violation, wrong dispatcher/queue
- 🟢 LOW — naming, style, dead code, unused import, optional polish

Prefix the title with **Nit:** (e.g. `🟢 Nit: Stranded comment left behind by the X removal`) when a finding is real but optional polish — a stale comment, a one-line leftover, something the author can take or leave without it blocking the PR. This is distinct from a plain 🟢 LOW finding that's still worth doing (e.g. a genuine unused import): Nit signals "skip this if you want," LOW signals "should probably fix."

Tone: findings, not verdicts. State the problem and its consequence; don't lecture. When something is a judgment call, say so ("Consider…" / "If X is intentional, ignore this"). Never pad the review with manufactured or speculative nitpicks to look thorough — a review with three real findings beats one with twenty trivia. A concrete, verifiable small catch (labeled Nit) is not padding and stays welcome.

### 8. Summary

After posting, print (heading depends on the resolved posting mode):

```
✅ Draft review saved (NOT submitted)          [Draft mode]
✅ Review posted — visible on the PR now       [Live mode]

Findings:
  🔴 Critical: <n>
  🟠 High:     <n>
  🟡 Medium:   <n>
  🟢 Low:      <n>

By category:
  🐛 Bugs/correctness:      <n>
  ⏳ Deprecated APIs:       <n>
  🧹 Code smells/dead code: <n>
  📐 Engineering standards: <n>
  🧪 Test coverage gaps:    <n>   (mobile-pr-test-analyzer)
  💬 Comment accuracy:      <n>   (mobile-pr-comment-analyzer — only if dispatched)
  🏗️  Type design:           <n>   (mobile-pr-type-design-analyzer — only if dispatched)

🔁 Duplicates skipped (already on PR): <n>

Review URL: https://github.com/<owner>/<repo>/pull/<number>

The review is pending. Open the PR in GitHub to inspect, edit,       [Draft mode]
or submit your comments when ready.
The review is live — comments are already visible on the PR.        [Live mode]
```

## Fallback (API call fails)

Do **not** fall back to `gh pr review --comment` or any other posting mechanism, in either mode. Instead, print the full findings to the terminal so the user can review and post manually if they choose:

```
❌ Could not create the review via API.
Error: <error message>

Findings are printed below for your reference.
Nothing was posted to GitHub.

<full findings report>
```
