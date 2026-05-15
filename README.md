# design-atlas

> Per-screen design research for SaaS, marketing sites, funnels, and mobile apps — powered by [Mobbin's MCP](https://docs.mobbin.com/mcp). A Claude Code skill.

One alignment interview → exhaustive sitemap → parallel Mobbin queries per screen → bundled HTML atlas with embedded feedback collection → locked design-system deck (tokens, components, wireframes, layers) ready to hand to a scaffolding step.

## What you get

For any product idea:

1. An **alignment interview** that locks artifact type (SaaS product / marketing site / funnel / mobile app / docs site), register, reference companies, anti-references, and per-screen genres before a single Mobbin query fires.
2. An **exhaustive sitemap** — every screen + every sub-state (empty / loading / error / populated / modal), categorized by surface, with explicit genre lock per screen so subagents query the right pattern.
3. A bundled **screen atlas HTML** — 4–8 references per screen, each with structured design-system observations (tokens, components, anti-patterns, lift-targets), light/dark toggle, and an embedded feedback collector you can triage in the browser.
4. A **hard feedback gate** — synthesis blocks until you paste back the exported feedback JSON. No auto-synthesis on unfiltered research.
5. A **component buffet** — per-category variant cards (button / input / card / chip / table-row / etc.) rendered as real DOM, not cropped JPEGs, with source-toggle for fidelity verification.
6. A **locked design-system deck** — tokens, typography scale, spacing/density, motion, iconography, ~9 core components with code snippets, 3–5 hero wireframes, **layers & interactions** (sidebars, slide-overs, modals, command palette, toasts, context menus), token export blocks, accessibility checklist.
7. A **pre-ship audit** — discriminator audit (every visual primitive at polish parity), asset verification (curl-test every URL, SVG path check, multi-width layout check), and a self-critical pass before the deck is shipped.

The output is rich enough that a sister scaffolding skill can generate a working Next.js project from it without further design decisions.

## Install

### 1. Add Mobbin's MCP to Claude Code

```bash
claude mcp add mobbin --scope user --transport http https://api.mobbin.com/mcp
```

Then in Claude Code: `/mcp` → `mobbin` → `Authenticate` (browser OAuth). **Restart Claude Code** so the new tools surface.

Confirm it's loaded:

```bash
claude mcp list | grep mobbin
# mobbin: https://api.mobbin.com/mcp (HTTP) - ✓ Connected
```

A paid Mobbin plan is required for full library access (621K+ screens, 142K+ flows from shipped apps, hand-curated and updated weekly).

### 2. Drop the skill into your project

This skill is a single `SKILL.md` (the agent prompt) plus a `template.html` (the atlas scaffold). Drop both into your Claude Code skills directory:

```bash
git clone https://github.com/arkbridge/design-atlas.git ~/.claude/skills/design-atlas
```

Or copy `SKILL.md` + `template.html` into wherever your project keeps Claude Code skills.

## Use

In any Claude Code session:

```
/design-atlas <product-name>
```

The skill takes over with a Phase 1 alignment interview, then enumerates screens, then dispatches parallel Mobbin queries, then opens the atlas in your browser for triage.

Other natural triggers:

- "design atlas for X"
- "screen atlas for X"
- "per-screen inspiration for X"
- "research every screen of X"

## Output

- `data/<product-slug>-screen-atlas.html` — full HTML atlas, light/dark toggle, embedded feedback collector
- `data/<product-slug>-screen-atlas.md` — terminal-friendly markdown summary
- `data/_atlas-work/` — scratch JSONs (alignment, sitemap, genres, per-bucket results, feedback, buffet selections) — keep these so the deck can be rebuilt without re-querying Mobbin
- `data/<product-slug>-design-system.html` — locked design-system deck after feedback gate clears

## Beyond SaaS

The workflow + 16 hard rules apply directly to:

- **SaaS product UI** (default)
- **Marketing websites** (Stripe.com / Linear.app / Vercel.com register)
- **Conversion funnels** (signup / paywall / checkout / onboarding)
- **Mobile apps** (iOS / Android)
- **Documentation sites** (docs.stripe.com / vercel.com/docs)

See the "Vertical Extensibility" section in `SKILL.md` for per-vertical genre vocabularies, component emphasis, and layer specimens.

## Why this exists

A naive "find me some references" workflow produces three failures every time:

1. **Pattern-level analyses, not token-level.** "This one has a sidebar" doesn't tell you what color the sidebar is, how tall the rows are, what the type stance is. So synthesis runs on prose and produces a watery system.
2. **No feedback loop before synthesis.** Anything you'd have rated "anti-pattern" still leaks into the locked design system.
3. **Design system + scaffold conflated in one pass.** Bundling code-generation into synthesis hides quality issues that would have surfaced in a separate verification step (e.g., dark-on-dark SVG text that's invisible).

This skill fixes all three: structured per-reference observations during research, in-HTML feedback collection with a hard gate before synthesis, and a design-system deck that ships ONLY the visual deck — scaffolding is a separate sister step.

## The 16 hard rules

Pulled from real end-to-end runs. Documented in detail in `SKILL.md`:

1. Mobbin image URLs expire — `curl` them during research, never reference shortlinks in the atlas
2. Feedback comes AFTER evidence — no rating questions before references are shown
3. Light/dark toggle is mandatory on every output artifact
4. Contrast guard before write — every SVG `<text>` gets an explicit `fill` attribute
5. Hard gates between phases (sitemap confirm, feedback paste, design-system review)
6. Discriminator audit before ship — every visual primitive at polish parity
7. Genre lock per screen — name the genre from a fixed vocabulary before designing
8. Asset-verification gate — curl every URL, render every SVG, no silent failures
9. Component buffet = real DOM, not cropped JPEGs
10. Iteration-count minimization is the KPI (not single-iteration speed)
11. Layout discipline — content dictates container (7 sub-rules)
12. Z-index discipline — fixed scale by purpose
13. Status-pill discipline — encode state once, not twice
14. Operator-dashboard structure — answer 4 questions in order
15. Toast-confirm pattern for mock UIs (no dead buttons)
16. Cheap-feeling UI smells — match canonical premium patterns

## Prerequisites

- **Claude Code** (the CLI)
- **Mobbin paid plan** + MCP authenticated
- A working directory with a `data/` folder the skill can write to

## License

MIT. See `LICENSE`.

## Credits

Built and battle-tested across real SaaS / marketing / funnel design work. The hard-rules section reflects every failure mode that shipped a re-do; the workflow is the shape that stopped those re-dos.

Mobbin's MCP and curated library make the per-screen-at-scale approach possible — see [docs.mobbin.com/mcp](https://docs.mobbin.com/mcp).
