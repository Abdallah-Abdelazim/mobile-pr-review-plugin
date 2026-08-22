# review-mobile-pr (skill)

Owns the PR-review orchestration logic (`SKILL.md`) and the platform review knowledge bases (`references/*.md`) that both `SKILL.md`'s own deprecation pass and the plugin's bundled agents read directly. Does NOT own agent definitions themselves — those live in `../../agents/` (see root `CLAUDE.md`).

## Entry Points

- `SKILL.md` - the orchestrator: pre-flight, platform detection, agent dispatch, aggregation, cross-check against existing PR comments, posting, summary
- `references/android.md`, `references/ios.md`, `references/kmp.md`, `references/engineering-excellence.md` - read directly by agents via their `Read` tool, given as absolute (or `${CLAUDE_PLUGIN_ROOT}`-relative) paths by `SKILL.md` — never pasted inline as excerpts

## Contracts & Invariants

- Every reference file is **literal review criteria an agent checks a diff against**, not documentation for leisurely reading. Terse checklist bullets only — no prose padding, no decorative headers. An agent cannot execute vague prose.
- `android.md` and `ios.md` each have a "Table of contents" that must stay in sync with their actual H2 headers — adding or renaming an H2 section without updating the TOC is a bug. `kmp.md` and `engineering-excellence.md` deliberately have no TOC (short enough not to need one) — don't add one just for consistency.
- The Deprecations tables in `android.md`/`ios.md` use an exact column format: `| Newly added usage of… | Status | Replacement / note | Severity |`. `mobile-pr-deprecation-scanner` (the agent) structurally depends on this shape — don't reflow it into prose.
- `engineering-excellence.md` "always applies, regardless of platform" — `SKILL.md` step 3 must always pass its path to `mobile-pr-code-quality-reviewer`, even on a PR that only touches one platform's files.
- Version/date-specific facts in these files (Android API level, iOS/Xcode/Swift version, Compose Multiplatform version) are **verified-live facts, not evergreen prose** — when editing, confirm current values via WebSearch rather than assuming last year's numbers still hold. These files get stale on their own schedule, independent of the code they describe.
- `SKILL.md`'s workflow steps are numbered 1–8 and cross-referenced by number from the Posting mode section, the Safety contract, and steps 3–6 themselves. Renumbering requires a repo-wide grep-and-fix, not a local edit.

## Patterns

Adding coverage for a new platform API/feature:
1. Find the matching H2 section in the relevant reference file (or add a new one if it's a genuinely new area — see `kmp.md`'s "Compose Multiplatform" section for an example of a whole new section added this way)
2. Write terse checklist bullets matching the surrounding style — problem + why it matters + the fix, not an essay
3. If a new H2 section was added to `android.md`/`ios.md`, update that file's Table of contents
4. Mirror the change into the sibling copy at `~/.claude/skills/review-mobile-pr/references/` (see root `CLAUDE.md`'s parity invariant) — verify with `diff` before committing

## Anti-patterns

- Don't duplicate a check that a specific agent already owns into these reference files' checklist bullets — test *coverage*/*quality* belongs to `mobile-pr-test-analyzer`'s own prompt, comment accuracy to `mobile-pr-comment-analyzer`'s, type-design invariants to `mobile-pr-type-design-analyzer`'s. These reference files are the shared platform-knowledge backstop (read by `mobile-pr-code-quality-reviewer` and the bug/silent-failure hunters), not a place to re-litigate what a specialist agent already checks better.
- Don't invent a deprecation/version claim to fill out a table row — an unverified "fact" here produces a false positive on every PR that touches the flagged API.

## Related Context

- Plugin packaging, agent parity rules, posting-mode contract: root `CLAUDE.md`
- Agent specs that consume these files: `../../agents/mobile-pr-*.md`
