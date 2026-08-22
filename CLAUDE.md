# mobile-pr-review (plugin repo)

A Claude Code plugin: one skill (`review-mobile-pr`) + 7 bundled subagents for expert Android/iOS/KMP PR review, packaged as its own installable marketplace.

## Intent Layer

**Before modifying code in a subdirectory, read its AGENTS.md first** to understand local patterns and invariants.

- **Review knowledge base & orchestration**: `skills/review-mobile-pr/AGENTS.md` - the skill itself (SKILL.md) and the platform review reference docs it and the agents read

No node for `agents/` or `.claude-plugin/` — both are below the token threshold; covered inline below.

```
mobile-pr-review-plugin/
├── .claude-plugin/
│   ├── plugin.json         # name/version/description — this repo IS the plugin
│   └── marketplace.json    # this repo is also its own single-plugin marketplace (source: "./")
├── agents/                  # 7 subagent specs — auto-namespaced mobile-pr-review:mobile-pr-* on install
└── skills/review-mobile-pr/ # the skill — see its AGENTS.md
```

## Key Invariants

- `plugin.json`'s `"name"` field must stay `mobile-pr-review` — it's baked into `marketplace.json`'s plugin entry, the README's install commands, and every `mobile-pr-review:mobile-pr-*` agent reference inside `skills/review-mobile-pr/SKILL.md`. Renaming it breaks all three.
- Every file in `agents/` here must stay byte-identical to its counterpart at `~/.claude/agents/mobile-pr-*.md` (the maintainer's personal global copies, used to test changes locally before they're packaged). Editing one without the other silently desyncs local testing from what ships. Same rule for `skills/review-mobile-pr/{SKILL.md,references/*.md}` vs. `~/.claude/skills/review-mobile-pr/{SKILL.md,references/*.md}` — except the two SKILL.md copies intentionally diverge on exactly two things: this repo's copy uses `mobile-pr-review:mobile-pr-*` namespaced agent names and `${CLAUDE_PLUGIN_ROOT}`-relative reference paths; the personal copy uses bare `mobile-pr-*` names and `~/.claude/skills/...`-absolute paths. Any other divergence between the two SKILL.md copies is a bug.
- This repo's `SKILL.md` implements a **draft-or-live posting mode** (defaults to draft/pending review; live posts immediately) resolved via a `--draft`/`--live` invocation flag or a one-time prompt. It's referenced from 6 places in that file (Posting mode section, Safety contract, step 1 header, step 7's API payload, step 8's summary, the Fallback section) — a change to this feature must be applied consistently at all 6, in both SKILL.md copies.
- No agent, and no step in `SKILL.md`, may use `gh pr review --comment`, `gh pr comment`, or the issues comments API — everything posts through the single `pulls/<number>/reviews` call in step 7, so draft and live comments always land together in one review.

## Patterns

Cutting a release, once the version-bump commit is pushed to `main`:
1. **The git tag is the plain semver string** — `<major>.<minor>.<patch>` (e.g. `1.3.0`), matching `plugin.json`'s `"version"` exactly. No `v` prefix, no plugin-name prefix. `git tag -a 1.3.0 -m "1.3.0"` then `git push origin 1.3.0`. (`claude plugin tag` defaults to its own `{name}--v{version}` format with no override flag — don't use it here; tag manually instead.)
2. `gh release create 1.3.0 --title "1.3.0" --notes "..."` — every tagged version gets a corresponding GitHub release, not just a bare tag

## Anti-patterns

- Don't add an 8th (or remove a) agent without also updating `SKILL.md`'s "Always dispatch" / "Dispatch conditionally" tables in steps 3 and 4 — the agent list there is the actual dispatch contract, not just documentation.
- Don't bump `plugin.json`'s version without checking whether `README.md`'s description/usage text is still accurate for what changed.
- Don't bump `plugin.json`'s version without also tagging and publishing a GitHub release for it (see Patterns above) — an untagged version bump has no corresponding release for `/plugin update` users or anyone browsing the repo's release history to land on.

## Related Context

- Review knowledge base & the skill itself: `skills/review-mobile-pr/AGENTS.md`
