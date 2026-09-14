# mobile-pr-review (plugin repo)

A Claude Code plugin: one skill (`review-mobile-pr`) that inlines all 7 of its review-pass prompts directly in `SKILL.md` — no separate agent files, no plugin dependency — packaged as its own installable marketplace.

## Intent Layer

**Before modifying code in a subdirectory, read its CLAUDE.md first** to understand local patterns and invariants.

- **Review knowledge base & orchestration**: `skills/review-mobile-pr/CLAUDE.md` - the skill itself (SKILL.md, including its inline review-pass prompts) and the platform review reference docs it reads

No node for `.claude-plugin/` — below the token threshold; covered inline below.

```
mobile-pr-review-skill/
├── .claude-plugin/
│   ├── plugin.json         # name/version/description — this repo IS the plugin
│   └── marketplace.json    # this repo is also its own single-plugin marketplace (source: "./")
└── skills/review-mobile-pr/ # the skill — see its CLAUDE.md
```

## Key Invariants

- `plugin.json`'s `"name"` field must stay `mobile-pr-review` — it's baked into `marketplace.json`'s plugin entry and the README's install commands. Renaming it breaks both.
- `skills/review-mobile-pr/{SKILL.md,references/*.md}` should stay in sync with any personal global mirror at `~/.claude/skills/review-mobile-pr/{SKILL.md,references/*.md}` the maintainer keeps for local testing before packaging — as of this writing no such mirror exists on disk, so there's nothing to sync today, but re-create it in sync if one is added later. (There is no longer an `agents/` ↔ `~/.claude/agents/` parity to maintain — the 7 review passes are defined only once, inline in `SKILL.md`'s "Review passes" section.)
- This repo's `SKILL.md` implements a **draft-or-live posting mode** (defaults to draft/pending review; live posts immediately) resolved via a `--draft`/`--live` invocation flag or a one-time prompt. It's referenced from 6 places in that file (Posting mode section, Safety contract, step 1 header, step 8's API payload, step 9's summary, the Fallback section) — a change to this feature must be applied consistently at all 6.
- `SKILL.md` also implements an opt-in **fix mode** (`--apply-safe-fixes`) that lets the orchestrator itself apply narrow, suggestion-block-grade fixes directly via `Edit`, instead of only posting comments — the only place in this skill anything ever touches the target repo's files. Every dispatched review pass stays read-only; only step 7 of `SKILL.md`'s workflow may edit, and only for a finding it re-verifies immediately beforehand.
- No review pass, and no step in `SKILL.md`, may use `gh pr review --comment`, `gh pr comment`, or the issues comments API — everything posts through the single `pulls/<number>/reviews` call in step 8, so draft and live comments always land together in one review.
- Every dispatched pass reports a `<confidence: HIGH/MEDIUM/LOW>` alongside its severity; step 5 drops LOW-confidence findings outright before cross-check. Don't remove the confidence field from a pass's output-format line without also updating step 5's filtering logic.
- Step 3 has a **tiny-diff exception**: for a genuinely small, low-risk diff, the orchestrator may review it directly instead of dispatching the full set of passes, as long as it still produces findings in the same output-format shape. This is a deliberate cost/latency shortcut, not a loophole to skip review rigor on anything with real logic.
- `SKILL.md`'s frontmatter has no `model:` field on purpose — the skill (and every pass it dispatches as a general-purpose agent) runs on whatever model the session already has selected. Don't re-pin one without a specific reason, and if you do, say why in the same commit.

## Patterns

Cutting a release, once the version-bump commit is pushed to `main`:
1. **The git tag is the plain semver string** — `<major>.<minor>.<patch>` (e.g. `1.3.0`), matching `plugin.json`'s `"version"` exactly. No `v` prefix, no plugin-name prefix. `git tag -a 1.3.0 -m "1.3.0"` then `git push origin 1.3.0`. (`claude plugin tag` defaults to its own `{name}--v{version}` format with no override flag — don't use it here; tag manually instead.)

That's it — **tag only, no GitHub release**. Don't run `gh release create`.

## Anti-patterns

- Don't add an 8th (or remove a) review pass without also updating `SKILL.md`'s "Always dispatch" / "Dispatch conditionally" tables in steps 3 and 4, its own prompt block in the "Review passes" section, and the summary table near the top — the dispatch tables are the actual dispatch contract, not just documentation.
- Don't bump `plugin.json`'s version without checking whether `README.md`'s description/usage text is still accurate for what changed.
- Don't bump `plugin.json`'s version without also creating the matching git tag (see Patterns above) — an untagged version bump has nothing for `/plugin update` or a future `git checkout <version>` to land on. Don't create a GitHub release for it, though — this repo's releases are tags only.
- **Don't add a new review check by writing it directly into one of `SKILL.md`'s inline pass blocks.** A real mistake, not a hypothetical: the redundant-state/efficiency/leaky-abstraction checks were first added straight into the Code-Quality Reviewer's `SKILL.md` block, duplicating content that belonged in `engineering-excellence.md` — the pass was then reading the same checklist twice (once inline, once from the file it's told to treat as "the concrete checklist you run"). New checklist substance belongs in the matching reference file's H2 section (see `skills/review-mobile-pr/CLAUDE.md`'s "Adding coverage" pattern); a pass's own `SKILL.md` block should stay limited to persona, process, and whatever is genuinely unique to that pass (not already covered by a reference file it reads).

## Related Context

- Review knowledge base & the skill itself: `skills/review-mobile-pr/CLAUDE.md`
