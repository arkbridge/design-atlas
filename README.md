# Design Atlas

Per-screen design research using Mobbin's MCP. One alignment interview, exhaustive sitemap, parallel Mobbin queries per screen, one bundled HTML deliverable.

## What it does

For any SaaS / app idea:

1. Interviews you about the product — register, brand anchors, must-have flows, anti-references.
2. Enumerates **every screen** the product will have, including sub-states (empty / loading / error / populated / modal).
3. Queries Mobbin's MCP per screen with multiple angles, curates 4–8 references each, writes specific analysis.
4. Bundles everything into a single self-contained HTML report (`screen-atlas.html`) with sidebar nav + per-screen sections.

## Usage

```
/design-atlas <product-name>
```

## Prerequisites

Mobbin's MCP must be installed and authenticated:

```bash
claude mcp add mobbin --scope user --transport http https://api.mobbin.com/mcp
# Then in Claude Code: /mcp → mobbin → Authenticate (browser OAuth)
# Restart Claude Code so the new MCP is visible
```

Paid Mobbin plan required for full library access.

## Output

- `data/<product-slug>-screen-atlas.html` — the full HTML report, open in browser
- `data/<product-slug>-screen-atlas.md` — terminal-friendly markdown summary

## See also

- `skills/design-atlas/SKILL.md` — full workflow instructions
- `skills/design-atlas/template.html` — HTML report template
- Related: `/design-lab <screen>` (Lazyweb skill) for follow-up UI variations per screen
