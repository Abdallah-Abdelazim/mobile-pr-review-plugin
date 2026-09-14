# mobile-pr-review-skill (repo)

An agent skill (`review-mobile-pr`) for expert Android/iOS/KMP PR review — one skill file with all 6 review-pass prompts inlined, no separate agent files. Installed with the `skills` CLI (`npx skills add ...`); see README.md.

## Intent Layer

**Before modifying code in a subdirectory, read its CLAUDE.md first** to understand local patterns and invariants.

- **Review knowledge base & orchestration**: `skills/review-mobile-pr/CLAUDE.md` - the skill itself (SKILL.md, including its inline review-pass prompts) and the platform review reference docs it reads

```
mobile-pr-review-skill/
└── skills/review-mobile-pr/ # the skill — see its CLAUDE.md
```

## Key Invariants

- `skills/review-mobile-pr/{SKILL.md,CLAUDE.md,references/*.md}` must stay in sync with the maintainer's personal global mirror at `~/.claude/skills/review-mobile-pr` (a symlink to `~/.agents/skills/review-mobile-pr`, used for local testing) — verify with `diff -rq` after any edit to this skill's files, and copy through if it drifts.
- This repo's `SKILL.md` implements an opt-in **`--lite` review mode** that narrows dispatch to just Bug Hunter + Code-Quality Reviewer (skipping Deprecation Scanner, Test Analyzer, Comment Analyzer, and Type-Design Analyzer entirely), on top of the tiny-diff exception which still takes precedence over it. It's referenced from 6 places in `SKILL.md` (the top summary table's Dispatch column, the "Review mode" section, the Usage block, step 1's header, step 3's dispatch tables + step 4's skip note, step 9's summary) — a change to this feature must be applied consistently at all 6.
- This repo's `SKILL.md` implements a **draft-or-live posting mode** (defaults to draft/pending review; live posts immediately) resolved via a `--draft`/`--live` invocation flag or a one-time prompt. It's referenced from 6 places in that file (Posting mode section, Safety contract, step 1 header, step 8's API payload, step 9's summary, the Fallback section) — a change to this feature must be applied consistently at all 6.
- `SKILL.md` also implements an opt-in **fix mode** (`--apply-safe-fixes`) that lets the orchestrator itself apply narrow, suggestion-block-grade fixes directly via `Edit`, instead of only posting comments — the only place in this skill anything ever touches the target repo's files. Every dispatched review pass stays read-only; only step 7 of `SKILL.md`'s workflow may edit, and only for a finding it re-verifies immediately beforehand.
- No review pass, and no step in `SKILL.md`, may use `gh pr review --comment`, `gh pr comment`, or the issues comments API — everything posts through the single `pulls/<number>/reviews` call in step 8, so draft and live comments always land together in one review.
- Every dispatched pass reports a `<confidence: HIGH/MEDIUM/LOW>` alongside its severity; step 5 drops LOW-confidence findings outright before cross-check. Don't remove the confidence field from a pass's output-format line without also updating step 5's filtering logic.
- Step 3 has a **tiny-diff exception**: for a genuinely small, low-risk diff, the orchestrator may review it directly instead of dispatching the full set of passes, as long as it still produces findings in the same output-format shape. This is a deliberate cost/latency shortcut, not a loophole to skip review rigor on anything with real logic.
- `SKILL.md`'s frontmatter has no `model:` field on purpose — the skill (and every pass it dispatches as a general-purpose agent) runs on whatever model the session already has selected. Don't re-pin one without a specific reason, and if you do, say why in the same commit.

## Anti-patterns

- Don't add a 7th (or remove a) review pass without also updating `SKILL.md`'s "Always dispatch" / "Dispatch conditionally" tables in steps 3 and 4, its own prompt block in the "Review passes" section, and the summary table near the top — the dispatch tables are the actual dispatch contract, not just documentation.
- **Prompt order is a cost lever, not incidental.** Every dispatched pass's prompt puts the shared PR context (intent, full diff, reference paths, output-format request) first, byte-identical across all passes, then that pass's own persona/checklist block last (see `SKILL.md` step 3 and the "Review passes" intro). This exists so the diff — the dominant token cost on a large PR — sits in a cacheable shared prefix instead of being duplicated per pass with no cache benefit. Don't flip the order back to "pass block, then context" for convenience.
- **Bug Hunter absorbs the error-handling lens** (formerly a separate Silent-Failure Hunter pass, merged 2026-09 to cut per-review dispatch count/cost on large diffs) — its prompt block runs both a correctness pass and a dedicated adversarial error-handling pass in one dispatch, with one merged severity scale. Don't split it back out into two passes without re-checking whether the cost tradeoff still favors doing so.
- **`--lite` mode's coverage gap is intentional, not a bug to patch.** Under `--lite`, nothing backstops deprecation checking or test-coverage checking — Deprecation Scanner and Test Analyzer just don't run, full stop. Don't give Code-Quality Reviewer (or Bug Hunter) an expanded, lite-mode-only scope to partially cover that gap — it was considered and rejected 2026-09: it would make a pass's own prompt block conditional on invocation flags (nothing else in this skill does that) and defeats the point of offering a cheaper mode in the first place.
- **Don't add a new review check by writing it directly into one of `SKILL.md`'s inline pass blocks.** A real mistake, not a hypothetical: the redundant-state/efficiency/leaky-abstraction checks were first added straight into the Code-Quality Reviewer's `SKILL.md` block, duplicating content that belonged in `engineering-excellence.md` — the pass was then reading the same checklist twice (once inline, once from the file it's told to treat as "the concrete checklist you run"). New checklist substance belongs in the matching reference file's H2 section (see `skills/review-mobile-pr/CLAUDE.md`'s "Adding coverage" pattern); a pass's own `SKILL.md` block should stay limited to persona, process, and whatever is genuinely unique to that pass (not already covered by a reference file it reads).

## Related Context

- Review knowledge base & the skill itself: `skills/review-mobile-pr/CLAUDE.md`
