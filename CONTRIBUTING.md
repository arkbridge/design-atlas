# Contributing to design-atlas

This skill works well for **SaaS product UI/UX** (operator dashboards, B2B app screens, internal tools, settings, modals). It does **not** yet work well for marketing websites, conversion funnels, mobile apps, or documentation sites — those verticals are marked `EXPERIMENTAL` or `UNTESTED` in the Vertical Extensibility section of `SKILL.md`.

PRs that **improve** the skill for *any* vertical are welcome. The bar:

## What we accept

1. **Real-world tested improvements.** The change must have been run on at least one real project, end-to-end, and produced a measurably better output. Theoretical improvements that haven't shipped are not enough. Include before/after evidence in the PR description — a screenshot of the output before your change vs after, or a description of the specific failure mode your change resolves.

2. **Vertical-specific Phase 5 prompt engineering.** If you've calibrated the synthesis subagent's prompts for a non-SaaS vertical (marketing-site, conversion-funnel, mobile-app, docs-site) and the resulting deck holds up across multiple runs, that's the kind of unlock this skill needs most. Document the prompt changes, the vertical they apply to, and the specific failure modes they avoid.

3. **New Hard Rules** — must be anchored by a real failure case the rule would have prevented. The "Why this rule exists" footer of every Hard Rule names a specific incident. If you can't name an incident, the rule isn't ready. Hard Rule 11.h (text-overflow audit) and Hard Rule 18 (source mix by artifact type) are both anchored to specific shipped failures — that's the bar.

4. **New phases or sub-phases.** Phase 5.25 (Asset Generation via Gemini Image) is an example. New phases should be opt-in by default, gated by an `alignment.json` flag, and fall back gracefully when the new behavior isn't applicable.

5. **Bug fixes** — text-overflow violations the audit misses, ref-card layout breaking at unusual viewport widths, atlas chrome that fails on a specific browser, accessibility regressions. Include the reproduction.

## What we don't accept (yet)

- **Theoretical rule additions** without a real failure-mode incident behind them. Hard Rules are a contract; we don't add them speculatively.
- **Style preferences** without evidence. If your change makes the output look different but not measurably better, it's a fork, not a PR.
- **Breaking changes** to Phase output schemas (alignment.json, results-*.json, feedback.json) without a migration path documented.
- **MCP-server replacements** for the existing Mobbin / Lazyweb integrations. Suggest extensions instead.

## How to submit

1. Fork the repo.
2. Make your changes. If you're adding a Hard Rule, add it to the numbered list with the same structure as the existing rules (rule statement → sub-rules if any → "Why this rule exists" anchor).
3. Test end-to-end on at least one real project. The skill is designed around one-shot runs (`/design-atlas <product>`); run it.
4. Open a PR. In the description, include:
   - **What changed** (one paragraph)
   - **What incident / failure mode this addresses** (specific, not theoretical)
   - **Before/after evidence** (screenshots, output JSON diffs, or a description of the visible improvement)
   - **What verticals this affects** (saas-product, marketing-site, etc)
   - **Any new opt-in flags** (default off, ideally)

## Discussion

Issues and proposals before code: open an issue first if you're planning a substantial change (new phase, new Hard Rule, vertical-specific prompt overhaul). Smaller fixes can go directly to a PR.

The maintainer reviews PRs against the rule "would this have caught a real failure I shipped, OR demonstrably improved a real output I shipped." Drive-by suggestions without that evidence will likely be closed with a request for more grounding.
