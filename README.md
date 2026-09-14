# mobile-pr-review

An [agent skill](https://github.com/vercel-labs/skills) for expert Android, iOS & Kotlin Multiplatform (KMP) pull request review. By default it saves every finding as a **PENDING (draft) GitHub review** — invisible until you manually submit it — but you can ask it to post live instead, either up front or when it asks you.

Reviews Kotlin/Jetpack Compose/Gradle, Swift/SwiftUI/UIKit, and KMP code — including shared **Compose Multiplatform** UI — against up-to-date (2026) platform deprecations, Swift 6 / Compose best practices, code smells, and software-engineering excellence standards.

## What's inside

- **1 skill** — `review-mobile-pr`, the orchestrator. Invoke it with a PR URL or number and it runs the whole review end to end.
- **6 built-in review passes**, defined inline in the skill and dispatched in parallel, each a self-contained specialist:

  | Pass | Focus |
  |---|---|
  | Bug Hunter | Correctness — forgotten call sites, unhappy paths, wrong/non-exhaustive logic, contract mismatches, concurrency bugs — plus a dedicated error-handling lens: swallowed exceptions, unjustified fallbacks, overly broad catches |
  | Code-Quality Reviewer | Code smells, dead/unused code, duplication, SOLID/naming/PR-scope, platform checklist backstop |
  | Deprecation Scanner | APIs deprecated/superseded/removed as of 2026 (Android 16/17 — API 36/37, Swift 6, iOS 17–26) |
  | Test Analyzer | Test coverage gaps and tests that don't exercise what they claim to |
  | Comment Analyzer | Comment/doc accuracy, stranded artifacts from incomplete deletions |
  | Type-Design Analyzer | Type encapsulation and invariant expression (Kotlin sealed classes/data classes, Swift structs/enums/protocols) |

Every pass reads a platform reference checklist matching the diff — `android.md`, `ios.md`, or `kmp.md` — the last of which includes a dedicated **Compose Multiplatform** section (shared Composables, `expect`/`actual` UI, CMP resources, cross-platform navigation, iOS `ComposeUIViewController` embedding).

No separate agent files and no external dependency beyond the GitHub CLI — everything this skill needs, including every review pass's prompt, ships inline in this one skill. For a genuinely small, low-risk diff (a typo fix, a comment-only edit), it may review directly instead of dispatching all 6 passes — same findings, less overhead.

Three cost tiers, from cheapest to most thorough: the tiny-diff shortcut above (zero dispatch, always wins when it applies), `--lite` (2 passes — Bug Hunter + Code-Quality Reviewer only, see below), and the full 6-pass dispatch (the default for anything that isn't tiny).

## Install

Install with the [`skills` CLI](https://github.com/vercel-labs/skills), which works across coding agents — swap `claude-code` below for your agent's identifier if you use a different one:

```
npx skills add https://github.com/Abdallah-Abdelazim/mobile-pr-review-skill/tree/main/skills/review-mobile-pr --agent claude-code
```

This drops the skill into `.claude/skills/review-mobile-pr/` in the current project (or pass `--global` to install it to `~/.claude/skills/` instead, available in every project). Restart your agent (or start a new session) so it picks up the new skill.

### Use it

```
/review-mobile-pr https://github.com/<org>/<repo>/pull/<number>
/review-mobile-pr <number>                     # when already inside the repo; asks Draft-or-Live before posting
/review-mobile-pr <number> --live              # skip the question — post live immediately
/review-mobile-pr <number> --draft             # skip the question — save as a pending review (the default anyway)
/review-mobile-pr <number> --lite              # cheaper — only Bug Hunter + Code-Quality Reviewer, skips deprecation/test/comment/type-design passes
/review-mobile-pr <number> --apply-safe-fixes  # also apply narrow, safe fixes directly instead of just commenting on them
```
Flags combine — `--lite --apply-safe-fixes` works together.

Requires the [GitHub CLI](https://cli.github.com/) (`gh`) authenticated against the target repo (`gh auth status`).

## Safety contract

- **Draft by default.** Unless you explicitly asked for Live — via `--live` or by answering the prompt — every comment is saved as part of a **pending** GitHub review, visible only to you until you open the PR and submit it yourself.
- **Live posts everything at once**, the moment the review finishes — still just comments, never an approval or a change request; this skill reports findings, it doesn't gate the PR.
- The skill never uses `gh pr review --comment`, `gh pr comment`, or any GitHub write API call that bypasses the single review it builds.
- If the API call fails, it prints the findings to your terminal instead of falling back to any other posting mechanism, in either mode.
- **The skill never edits your repo's files, unless you explicitly pass `--apply-safe-fixes`.** Even then, it only ever auto-applies a narrow class of unambiguous, single-line-grade fixes — anything else still becomes a review comment for you to act on yourself.
- **`--lite` is a coverage tradeoff, not free.** With it, deprecation checking, test-coverage checking, and comment/type-design review don't happen at all — only correctness, error-handling, and code-quality/hygiene do.

## License

MIT — see [LICENSE](./LICENSE).
