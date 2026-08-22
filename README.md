# mobile-pr-review

A [Claude Code](https://claude.com/claude-code) plugin for expert Android & iOS pull request review. It saves every finding as a **PENDING (draft) GitHub review** — nothing is ever posted publicly until you manually submit it.

Reviews Kotlin/Jetpack Compose/Gradle, Swift/SwiftUI/UIKit, and Kotlin Multiplatform (KMP) code against up-to-date (2026) platform deprecations, Swift 6 / Compose best practices, code smells, and software-engineering excellence standards.

## What's inside

- **1 skill** — `review-mobile-pr`, the orchestrator. Invoke it with a PR URL or number and it runs the whole review end to end.
- **7 dedicated subagents**, dispatched in parallel, each a self-contained specialist:

  | Agent | Focus |
  |---|---|
  | `mobile-pr-bug-hunter` | Correctness — forgotten call sites, unhappy paths, wrong/non-exhaustive logic, contract mismatches, concurrency bugs |
  | `mobile-pr-silent-failure-hunter` | Swallowed exceptions, unjustified fallbacks, overly broad catches |
  | `mobile-pr-code-quality-reviewer` | Code smells, dead/unused code, duplication, SOLID/naming/PR-scope, platform checklist backstop |
  | `mobile-pr-deprecation-scanner` | APIs deprecated/superseded/removed as of 2026 (Android 16/API 36, Swift 6, iOS 17–26) |
  | `mobile-pr-test-analyzer` | Test coverage gaps and tests that don't exercise what they claim to |
  | `mobile-pr-comment-analyzer` | Comment/doc accuracy, stranded artifacts from incomplete deletions |
  | `mobile-pr-type-design-analyzer` | Type encapsulation and invariant expression (Kotlin sealed classes/data classes, Swift structs/enums/protocols) |

No third-party plugin dependency — every agent this skill needs ships in this repo.

## Install

### 1. Add the marketplace

```
/plugin marketplace add Abdallah-Abdelazim/mobile-pr-review-plugin
```

### 2. Install the plugin

```
/plugin install mobile-pr-review@mobile-pr-review
```

Restart Claude Code (or start a new session) so it picks up the new skill and agents.

### 3. Use it

```
/review-mobile-pr https://github.com/<org>/<repo>/pull/<number>
/review-mobile-pr <number>   # when already inside the repo
```

Requires the [GitHub CLI](https://cli.github.com/) (`gh`) authenticated against the target repo (`gh auth status`).

## Safety contract

- Nothing is posted publicly — every comment is saved as part of a **pending** GitHub review, visible only to you until you open the PR and submit it yourself.
- The skill never uses `gh pr review --comment`, `gh pr comment`, or any GitHub write API call that posts immediately.
- If the pending-review API call fails, it prints the findings to your terminal instead of falling back to any public posting mechanism.

## Installing this repo's plugin for others

Anyone can pick up this plugin the same way:

```
/plugin marketplace add Abdallah-Abdelazim/mobile-pr-review-plugin
/plugin install mobile-pr-review@mobile-pr-review
```

To update to a newer version after a push, run `/plugin marketplace update mobile-pr-review` then reinstall, or use `/plugin update` if your Claude Code version supports it.

## License

MIT — see [LICENSE](./LICENSE).
