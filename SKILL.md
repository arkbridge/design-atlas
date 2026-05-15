# Design Atlas — v2

<command-name>design-atlas</command-name>

## Description

Take a product idea, run an alignment interview, enumerate every screen the SaaS will need, query Mobbin's MCP per screen for tailored inspiration WITH structured design-system observations, and produce a single bundled HTML atlas with **embedded per-reference feedback collection**. After the user reviews and submits feedback, synthesize a locked design-system deck (tokens + components + wireframes). The scaffold step (turning the design system into a working clickable+animated localhost frontend) lives in a sister skill that runs only after the design-system deck is reviewed.

The output unit of research is the **screen**. Each screen accumulates 4–8 references with structured observations the synthesis stage runs on.

## Triggers

- `/design-atlas <product-name>`
- "design atlas for X"
- "screen atlas for X"
- "per-screen inspiration for X"
- "research every screen of X"

## Why v2 (changes from v1)

v1 produced an atlas + design system in one pass with no feedback gate. Three problems surfaced:

1. **Reference analyses were screen-pattern level**, not design-system-token level. Synthesis ran on prose, not structured observations — produced a watery system.
2. **No feedback loop.** Synthesis used the unfiltered research. Anything the user would have rated "anti-pattern" still leaked in.
3. **Design system + scaffold conflated.** Bundling code-generation into synthesis hid quality issues (e.g. invisible-text contrast bugs that would have surfaced in a separate verification step).

v2 fixes all three: structured observations during research, in-HTML feedback collection with hard gate, design-system synthesis that ONLY produces the visual deck, scaffold deferred to a sister skill.

## Hard Rules (do NOT violate)

These rules emerged from end-to-end runs. Any future run that violates one of them will reproduce a known bad output.

1. **Mobbin image URLs expire — save locally during Phase 3.** The `image_url` returned by `mcp__mobbin__search_screens` is an MCP-session-scoped shortlink (`https://mobbin.com/api/mcp/short/...`) that returns HTTP 410 Gone after the session ends. **But freshly-fetched shortlinks return HTTP 200** — only previously-stored ones expire. So the working pattern is: in the SAME agent session that runs the Mobbin search, immediately `curl -L -o {local_path} "{fresh_image_url}"` via Bash. (Mobbin's JSON response does NOT expose base64 — it's only delivered as a model-visible attachment, not a programmatic field. So curl on the fresh URL is the correct path, not base64 decode.) Save as `data/_atlas-work/ref-images/{bucket}-{screen-id}-{ref_idx}.jpg`, update each ref's JSON with `local_image_path`. The atlas template uses `<img class="thumb" loading="lazy" src="{local_image_path}">` — NEVER `style="background-image:url(image_url)"`. Atlas with original Mobbin URLs = broken thumbnails on second open.

2. **Feedback comes AFTER evidence — locked order.** Per-screen order is `substates → references grid → recommended pattern → direction radio + steer textarea`. Per-surface order is `screens → approach textarea`. Asking the user to give a "direction" or rate a "suggestion" before they've seen the references is a UX bug. Global register steer can sit at top because it's pre-evidence high-level direction; everything else: feedback after evidence, no exceptions.

3. **Light/dark mode toggle is mandatory on ALL output artifacts.** Atlas HTML, design-system deck, scaffold's running app — all three must define `:root` (light default) + `[data-theme="dark"]` (dark override) token sets, include an early `<head>` script that sets `data-theme` before paint to prevent flash-of-wrong-theme, render a sidebar/header toggle button with `aria-pressed`, persist choice via `localStorage`, and auto-detect via `prefers-color-scheme` on first visit. For atlas/deck dark mode use a warm coffee/sepia register (cream-on-coffee), NOT the product's near-black register. Single-register output = fail.

4. **Contrast guard before write.** For ANY SVG that renders inside a non-default-background container, walk every `<text>` element and assert it has an explicit `fill=` attribute. Inheriting from a global `svg text { fill: var(--ink) }` rule on a dark background renders dark-on-dark = invisible. Build-time fail.

5. **Hard gates between phases.** Phase 2→3 (sitemap confirm), Phase 4→5 (feedback JSON paste), Phase 5→6 (design-system review). Skipping any of these = guaranteed re-do.

6. **Discriminator audit before ship.** Before declaring any UI surface done, walk EVERY visual primitive and verify polish parity:
   - Buttons + their states (hover, focus, disabled)
   - Cards + hover treatment + layered shadows
   - Charts (gradient fills, inner shines)
   - Step badges + connector lines
   - Progress bars + meters (gradient fill + shimmer)
   - Connector/brand tiles + logos
   - Dividers + separators
   - Numbered indicators / pills / chips
   - Icons + their containers
   - Empty states + skeletons + loading shimmers
   - Form inputs + focus rings
   - Status dots + pulse animations
   - Background gradients + inset highlights
   
   If ANY primitive is at default-CSS quality (flat solid color, 1px border, no shadow, no hover affordance), the surface FAILS the audit and gets fixed before showing the user. **This rule exists because polish trickle-down failed twice in early builds:** charts flat while cards polished, then progress bars flat while buttons polished. Same failure mode, both preventable.

7. **Genre lock as Phase 1 question.** Before designing any surface, name its genre EXPLICITLY from a fixed vocabulary BEFORE picking the layout. Acceptable genres:
   - `briefing-document` — narrative prose, dated, executive-summary on top (FT lead-story register)
   - `dashboard` — tile cards, KPIs, glanceable multi-source aggregation
   - `ledger-table` — dense tabular data, sortable, filterable, hover-row, audit-trail aesthetic
   - `activity-feed` — reverse-chronological log, timestamped, single-actor-per-line
   - `document-archive` — searchable + filterable list of static artifacts
   - `inbox-triage` — urgent-first, accept/modify/snooze/reject per item
   - `marketplace` — grid of cards + search + category sidebar
   - `family-tree` — hierarchical, SVG parent→child connectors, organizational shape
   - `timeline` — git-history-style event stream with typed dots
   - `canvas` — freeform spatial layout, draggable nodes
   - `wizard` — linear N-of-M step flow with progress bar
   - `composition` — real DOM components on styled background (design lab itself)
   
   Genre lock prevents the "briefing-document-as-tile-cards" failure (default dashboard pattern was wrong because the surface was actually a briefing-document) and the "family-tree-as-list" failure (correct genre was hierarchical, not flat). **Default-genre-by-vibe is forbidden. Name it, lock it, then design.**

8. **Asset-verification gate.** Anything that can fail silently must be visually verified before "done":
   - External image URLs (`<img src>`) → curl-test for HTTP 200 + non-empty body
   - External fonts → load in browser, confirm typography renders
   - SVG paths and viewBox math → render visually, never trust syntax-validity alone
   - Brand logos / 3rd-party assets → spot-check 3-5 to confirm they render
   - `backdrop-filter`, `mask`, modern CSS → test in target browsers, not just Chrome
   - Animations / transitions → open in browser, confirm motion fires
   
   The audit pattern:
   ```bash
   for url in $(grep -oE 'https://[^"]+' output.html | sort -u); do
     code=$(curl -sI -o /dev/null -w "%{http_code}" "$url")
     echo "$url → $code"
   done
   ```
   
   **This rule exists because of silent-failure incidents:** SVG paths with `%` units (rendered nothing) + brand-logo URLs that 404'd as broken-image placeholders. Both passed every non-visual check.

9. **Component buffet = real DOM, not cropped JPEGs (Path B).** Buffet picker variants MUST be rendered as live HTML/CSS components, not images of components. See Phase 4.75 below for the workflow.

10. **Iteration-count minimization is the KPI.** The success metric is "rounds-of-feedback per project," NOT "speed of single iteration." Slower upfront work with hard gates ships faster overall. Pre-ship self-critical pass is mandatory: re-read every output as if you were the user reviewing it, scanning for gaps the user would flag. If you would correct it as the user, correct it before showing.

11. **Layout discipline — content dictates container.** Every UI surface renders at its target viewport width. Preview containers narrower than target = horizontally scrollable, NEVER compressing content. The 7 sub-rules:
    
    a. **Native-dimension render** — components render at their design target width; deck containers scroll
    b. **Min-width discipline** — every card/panel/button row gets explicit `min-width` based on longest content; `1fr` is BANNED for content with minimum viable size
    c. **Fixed-pixel grid columns for variants** — `repeat(N, MINPX) justify-content: center`, never `repeat(N, 1fr)`
    d. **Button row safety check** — compute `(button1.text_width + gap + button2.text_width + padding)`; if it exceeds container, stack vertically or shrink to icon-only
    e. **Longest-content measurement at build time** — use a `long_label_safe_width(labels, ...)` helper to compute container min-width from longest realistic label
    f. **`overflow: hidden` is opt-in** — default is `visible` or `auto`; never silently clip content
    g. **Multi-width pre-ship check** — open every UI at native width AND at deck's content-column width; if anything looks crushed at narrower width, fix the container (scroll) not the content (don't shrink)
    
    Plus modal-button alignment: action buttons in modals render CENTERED (`justify-content: center`), not right-aligned.
    
    **Code-level enforcement:** Use layout primitives (`variant_grid()`, `button_row()`, `native_scrollable()`, `long_label_safe_width()`) instead of ad-hoc grid/flex CSS. These make violating the rule HARDER than following it.

12. **Z-index discipline — fixed scale by purpose.** Persistent fixed-position chrome (FAB, sticky CTAs, mobile bottom bars) MUST be at z-index 200 (fixed-tier) — BELOW modal/overlay layers. Modal=400, popover=500, tooltip=600, toast=700. Persistent chrome punching through overlays is always a bug. Plus: when the same overlay can be opened from multiple triggers (toolbar pill, keyboard shortcut, in-page button), lift state to ONE shared context — triggers consume `open()`, never own state. Also: sidebars / left rails / persistent right rails use `position: sticky; top: 0; height: 100vh` — they don't scroll with content.

13. **Status-pill discipline — encode state once, not twice.** When displaying state across many items in a grid/list, encode state with ONE color signal per item (border-accent, dot, or row-leading bar) + ONE legend strip explaining colors at the top. NEVER repeat the text label inside every item card ("HEALTHY", "SYNCING" repeated on every row = visual noise + uneven heights when labels wrap). Linear / Stripe / Notion all use this pattern. Plus: activity-feed timestamps go on the RIGHT (after the message, mono tabular-nums), not on the left.

14. **Operator-dashboard structure — answer 4 questions in order.** A primary dashboard for an operator role (CEO, founder, ops lead) must answer 4 questions, each with its own surface section, in this order: (1) **What's burning?** = Outstanding KPI tile + right-rail Quick Attention; (2) **What's moving?** = KPI strip of 4-6 tiles with one-glance numbers + deltas; (3) **What do I need to decide today?** = operator action panel (pending decisions, accept/review affordances); (4) **What's coming?** = today's calendar / schedule panel. The hero brief is ~30% of vertical space, NOT 70-100%. Right rail is ALWAYS sticky. Layout: KPI row → narrative middle → 3-panel operator action row → sticky right rail throughout.

15. **Toast-confirm pattern for mock UIs.** In any pre-backend or mock prototype, every clickable element gets one of: **Real action** (handler) · **Toast confirm** (`useToast()` with semantic kind/title/body explaining what WOULD happen) · **State toggle** (local useState + visual active-state) · **Modal stub** (multi-step flow opens Modal with form + final toast on submit) · **Remove** (delete the affordance entirely if no fit). Dead buttons are worse than no buttons — clicking with no feedback erodes trust in everything else. Batch toast wirings as the LAST pass over a built UI, not during build.

16. **Cheap-feeling UI smells — match canonical premium patterns.** Three smells to detect + fix:
    - **Sparse topbar** (just title + 2 buttons floating) → **workspace-shell pattern**: workspace switcher chip + dividers between functional groups + wider command-palette pill + avatar dropdown
    - **Colored single-letter notification icons** → **type-led design**: small colored dot + KIND label in mono + URGENT items get 3px red left-accent bar + grouped sections (NEEDS YOU / ACTIVITY)
    - **Two-row filter UI** (full-width search + grey ▾ buttons + dashed "+ add filter" chip) → **Linear-style unified single-row filter bar**: `[search] [+ Filter dropdown] [applied chips inline] [→spacer] [Sort ▾] [Density ▾]` with reusable `<FilterBar>` component
    
    Pattern: when ANY UI feels "cheap," ask (a) what's the canonical premium pattern for this surface (Linear / Vercel / Stripe / Notion all reference) (b) what's the SPECIFIC visual smell (sparse / variable-width / orphaned / floating) (c) does it match the design vocabulary already in the app (d) is it reusable enough to lift to a component.

17. **The atlas itself ships with the same polish as the design system it produces.** The Phase 4 atlas HTML is a working artifact the user actively triages — not scaffolding to throw away. All `ui-design-system` rules apply to the atlas chrome and feedback widgets, not just to the deck and scaffold. Specific sub-rules:

    a. **Ref-card grid MIN width = 460px.** `repeat(auto-fill, minmax(460px, 1fr))`. A 320px grid crushed the 5-chip rating row into 2-line wraps and clipped placeholder text on narrow viewports. 460px gives 2-column max at typical desktop widths and lets the rating row + notes breathe.

    b. **Rating chips are SINGLE-WORD pills, never label + description stacks.** Five chips fit one row at 460px ONLY if each is single-word. Per-chip explanation lives in (i) the always-visible help banner at the top of the page AND (ii) hover `title` tooltips. Stacking `<label>` + `<description>` inside each chip wraps unevenly (e.g. `LIFT MOVES` wraps to 2 lines while `OK` stays at 1, producing uneven chip heights).

    c. **Notes inputs are full-width, stacked vertically with labels above — never inline with labels on the left.** Inline labels eat 60-80px of input width on narrow cards, clipping placeholder text. Stacked = full-width input + tiny mono UPPERCASE label above.

    d. **Fieldset legends are ≤6 words, OR the question is moved OUTSIDE the fieldset.** A 14-word legend like `Decision: how does this reference inform [product]'s "Sticky nav / progress rail" section?` breaks the fieldset border, overflows the visible width, and reads worse than a short legend + a separate `<div class="fb-prompt">` above the fieldset. Pattern: `<div class="fb-prompt">How does this inform <strong>SectionName</strong>?</div><fieldset><legend class="visually-hidden">Reference rating</legend>...</fieldset>`.

    e. **Inputs use `font-size: 16px` minimum** to prevent iOS auto-zoom on focus. Placeholder text can be smaller (14px) but the input itself must be ≥16px.

    f. **Touch targets ≥36px on chips/buttons, ≥40px on inputs.** Even on desktop — keeps hit areas forgiving.

    g. **Always-visible help banner with the rating legend.** Top of page, 5-cell grid showing each chip's meaning. Not collapsible, not hidden — the user shouldn't have to scroll back up to remember what "HERO" means.

    h. **`color-scheme` meta + flash-prevention script** required (already in Hard Rule 3, but reinforced here).

    i. **`transition: all` is banned** in atlas CSS too. Specific properties only (`transition: color 0.12s, border-color 0.12s, background-color 0.12s`).

    j. **`tabular-nums` (`font-variant-numeric: tabular-nums`) on every numeric counter** — export-bar stats, ref counts, etc.

    k. **`role="radiogroup"` + `aria-label` on the rating row** for assistive tech; visible `<label>` per chip; hidden radio inputs (whole chip is the click target).

    l. **The framing question must name the SECTION explicitly:** "How does this inform `<section-name>`?" Generic "Rate this reference" leaves the user guessing whether they're rating the screenshot's quality vs its applicability to the design system. Always: applicability TO the named section.

    **Why this rule exists:** An early atlas iteration shipped with 5-chip stacked-description widgets at minmax(320px, 1fr) grid and received clear UX feedback that the layout was confusing and the framing question wasn't clear about what was being decided. Both complaints — layout AND framing — are preventable by applying ui-design-system rules to the atlas chrome from the start, not just to the design-system deck downstream.

---

## Vertical Extensibility — beyond SaaS products

This skill's workflow + rules apply DIRECTLY to:
- **SaaS product UI** (default framing — internal tools, B2B apps, operator dashboards)
- **Marketing websites + landing pages** (Stripe.com, Linear.app, Vercel.com style)
- **Conversion funnels** (signup flow, paywall, checkout, onboarding)
- **Mobile apps** (iOS / Android product surfaces)
- **Documentation sites** (docs.stripe.com, vercel.com/docs)

**How to adapt the workflow for non-product verticals:**

| Phase | Default (SaaS) | Marketing site / Funnel adaptation |
|---|---|---|
| **Phase 1 alignment** | "register / mood / brand anchors" | Add: conversion goal · target persona · funnel position · primary CTA |
| **Phase 1 reference companies** | Linear, Vercel, Stripe Atlas | Add: Stripe.com (marketing), Linear.app (marketing), Notion.so/product, Cal.com landing, Apple Vision Pro page |
| **Phase 2.5 genre lock** | briefing-document, dashboard, etc. | Add: hero-landing · pricing-page · feature-page · testimonial-wall · funnel-step · email-capture · checkout-step · post-conversion-thank-you · long-form-sales-letter · interactive-demo · blog-post · changelog |
| **Phase 4.75 component buffet** | button, input, card, chip, etc. | Add: hero-CTA, sticky-conversion-bar, social-proof-row, pricing-tier-card, testimonial-quote-card, FAQ-accordion, comparison-table, feature-highlight, video-embed, scroll-triggered-modal |
| **Phase 5.5 layers** | help/chat sidebar, slide-over, modal, command palette, etc. | Add: cookie-banner · exit-intent-popup · sticky-bottom-CTA · scroll-progress-bar · live-chat-widget · upsell-overlay · video-lightbox |

**The hard rules (1-17) all transfer unchanged.** Premium polish, discriminator audit, genre lock, asset verification, real-component buffet, iteration minimization, layout discipline, z-index discipline, status-pill discipline, operator-dashboard structure (where applicable), toast-confirm pattern, cheap-feeling UI smell detection, atlas-as-first-class-artifact — all apply equally to landing pages, funnels, and marketing sites.

**The reusable code primitives all transfer:** `variant_grid`, `button_row`, `native_scrollable`, `long_label_safe_width`, the React components (FilterBar, Topbar, DetailPanel, Modal, Dropdown, Toast) — built generically enough to drop into any web project.

**Cross-vertical anti-patterns** (universal — applies everywhere):
- Generic stock-photo hero (use real product screenshots or branded imagery)
- "Lorem ipsum" placeholder copy that ships to production
- Free fonts that look like Geist (use Geist itself, or commit to a real licensed display face)
- Dead CTAs that scroll the page instead of converting (every CTA must do something)
- Cookie banners that are 60% of the viewport
- Sticky bars that overlap content without offset
- Any element that looks like an ad even if it's not

## Prerequisites

### 1. Mobbin MCP loaded in current session

```bash
claude mcp list | grep mobbin
```

Expected: `mobbin: https://api.mobbin.com/mcp (HTTP) - ✓ Connected`

If `Needs authentication` → user types `/mcp` → mobbin → Authenticate (browser OAuth).

If not registered: `claude mcp add mobbin --scope user --transport http https://api.mobbin.com/mcp`, then **restart Claude Code**.

### 2. Mobbin tools surfaced

Confirm via `ToolSearch` query `mobbin` — if no `mcp__mobbin__*` tools appear, the MCP isn't loaded. Stop and tell the user to restart.

## Workflow

### Phase 1 — Alignment Interview

Use `AskUserQuestion` for clickable preference questions. Pre-fill from any persistent memory you have when the target product is known.

**Q0. ARTIFACT TYPE — REQUIRED, ASK FIRST, GATES EVERYTHING ELSE**

Before any other context-gathering, ask via `AskUserQuestion` what kind of artifact is being designed. The answer drives reference companies suggested, genre vocabulary used, buffet categories, layer specimens emphasized, and even the Phase 5.5 layers shipped.

Acceptable values (single-select):

| Type | Examples | Phase 2.5 genres weighted | Phase 4.75 component buffet emphasis | Phase 5.5 layers emphasis |
|---|---|---|---|---|
| `saas-product` | Internal tools, B2B SaaS, operator dashboards | dashboard, ledger-table, briefing-document, inbox-triage, family-tree, wizard | nav-rail, KPI tile, table row, status pill, command palette | help/chat sidebar, slide-over, modal, dropdown, command-palette, toast |
| `marketing-site` | Stripe.com, Linear.app, Vercel.com, Anthropic homepage | hero-landing, feature-page, pricing-page, testimonial-wall, blog-post, changelog | hero-CTA, social-proof-row, pricing-tier-card, testimonial-card, FAQ-accordion, comparison-table | sticky-bottom-CTA, exit-intent popup, video lightbox, scroll-progress bar |
| `conversion-funnel` | Signup flow, paywall, checkout, lead-gen audit form | funnel-step, email-capture, checkout-step, post-conversion-thank-you, paywall, upsell-overlay | step-indicator, form-field, plan-tier-selector, payment-form, urgency-banner | upsell overlay, trust-badge tooltip, abandoned-cart toast, social-proof live-counter |
| `mobile-app` | iOS/Android product UI, consumer app shells | tab-bar, native-modal, bottom-sheet, pull-to-refresh, gesture-driven view | nav-tab, list-row, card swipeable, modal-from-bottom, action-sheet | bottom-sheet, action-sheet, share-sheet, push-permission prompt |
| `docs-site` | docs.stripe.com, vercel.com/docs | doc-page, API-reference, code-example, sidebar-nav | TOC sidebar, code block, callout, search results, version selector | search command, copy-code toast, prev/next nav |
| `hybrid` | Product + marketing in same domain | varies — drives a per-section answer | varies | varies |

**Confirm the type, then suggest reference companies appropriate to it** in the next question (Q3a below). Don't suggest Linear/Vercel for a checkout funnel — suggest Stripe Checkout, Calendly booking, Notion's pricing page upgrade flow, etc.

**Once the artifact type is locked**, proceed with the rest of Phase 1:

1. **Product summary** — one-sentence what it is + one-sentence who it's for
2. **Register / mood** — Bloomberg-dense / Linear-minimal / Cal.com-friendly / Notion-spatial / custom (vocabulary varies by artifact type)
3. **Brand anchors** — color hint, type stance (serif/grotesque/mono), motion appetite (still/subtle/kinetic), info density (sparse/balanced/dense)
3a. **REFERENCE COMPANIES TO EMULATE — REQUIRED, NOT OPTIONAL** — block proceeding without 3-5 named anchors. Suggested examples vary by artifact type:
    - `saas-product` → Linear, Vercel, Stripe Atlas, Cursor, Arc, Notion premium, Linear Inbox, Pylon, Height, Raycast
    - `marketing-site` → Stripe.com, Linear.app, Vercel.com, Cal.com, Notion's marketing site, Apple Vision Pro page, Anthropic homepage
    - `conversion-funnel` → Stripe Checkout, Calendly booking flow, Vercel signup, Linear's onboarding, ConvertKit landing, Webflow pricing
    - `mobile-app` → Cash App, Linear mobile, Notion iOS, Things 3, Bear, Cron, Fey
    - `docs-site` → docs.stripe.com, vercel.com/docs, react.dev, Tailwind docs, Anthropic docs
    Anchoring direction from the start prevents the bland-default round. Pre-fill from persistent memory if a prior pick exists for known products. **DO NOT proceed to Phase 2 without this signal locked.**
4. **Must-have flows** — 3–5 core user journeys
5. **Anti-references** — what you do NOT want it to look like
6. **Mobbin scope** — `all` apps or scoped category? Mobile, desktop, or both? (auto-bias by artifact type — funnels skew mobile-friendly, SaaS skews desktop-primary)
7. **Output location** — default `data/<product-slug>-screen-atlas.html`

### Phase 2 — Screen Enumeration

Draft the **complete** sitemap. Every screen + every meaningful sub-state. Categorize by surface (auth, core workspace, creation flows, modals, settings, marketing, edges, mobile variants if scoped).

For each screen capture **3 sub-states minimum**: `empty | loading | populated`. Add `error | success | hover | selected | modal-open` where they materially differ.

Output as markdown checklist. **Get user confirmation BEFORE Phase 3** via `AskUserQuestion` — burning Mobbin queries on the wrong screen list is the worst failure mode.

### Phase 2.5 — Genre Lock per Surface (HARD GATE)

For EACH approved screen, name its genre from the fixed vocabulary BEFORE Phase 3 dispatches research subagents. Without an explicit genre lock, subagents will fetch references at the default-genre level (dashboard tiles, list rows) which causes the wrong-genre failure mode.

For each screen, fill in: `screen-id → genre`. Genres: `briefing-document | dashboard | ledger-table | activity-feed | document-archive | inbox-triage | marketplace | family-tree | timeline | canvas | wizard | composition`.

If a screen ambiguously fits two genres, use `AskUserQuestion` to lock the choice. Do NOT default-guess.

Example genre mappings for a generic SaaS product:
- `org-pulse` → briefing-document
- `decision-archive` → ledger-table
- `audit-timeline` → timeline
- `connector-marketplace` → marketplace
- `agent-org-chart` → family-tree
- `directives` → inbox-triage
- `reports-archive` → document-archive
- `tenant-setup` → wizard

Save to `data/_atlas-work/genres.json`. Pass into Phase 3 so subagents query Mobbin for references that match the genre, not the default pattern.

### Phase 3 — Per-Screen Research with Structured Observations

For each approved screen, parallel-dispatch subagents (one per surface bucket, max 6 in parallel). Each subagent does:

1. **Generate 1–3 query angles per screen** — different vocabularies targeting the same screen. Example for "billing settings": "billing settings subscription management" / "plan upgrade SaaS" / "invoice history payment method".

2. **Run Mobbin MCP queries** — `mcp__mobbin__search_screens` with `platform: "web"`, `mode: "fast"`, `limit: 8`. Cap results, dedupe across angles. **Discard the inline base64 image data** in working notes — keep only `app_name`, `flow_name`, `mobbin_url`, `image_url`.

3. **Curate 4–8 references per screen.** Selection criteria: source-app quality, register fit, app diversity (no more than 1–2 refs from same app), honor anti-references.

4. **Per-reference output (TWO blocks):**

   **Block A — screen-pattern analysis** (2–3 sentences specific to this product's version of this screen). Generic praise FORBIDDEN.

   **Block B — design-system observations** (structured JSON):

   ```json
   {
     "tokens_observed": {
       "row_height_estimate": "28px (dense data table)",
       "label_treatment": "11px UPPERCASE mono, ~0.16em letter-spacing, dim text",
       "accent_usage": "single warm-amber, only on row IDs and primary CTA",
       "icon_style": "stroke 1.5px, geometric",
       "type_register": "grotesque body + mono labels + serif headings",
       "dominant_palette_estimate": ["#0e0f12 (page bg)", "#16191d (panel)", "#a3a9b1 (text-2)", "#e8b04c (accent)"],
       "border_treatment": "1px hairline, no radius",
       "spacing_rhythm": "16/24/32 — feels 4-based"
     },
     "components_observed": [
       "data-table — sticky thead, monospace ID column, no row borders, hover lift +4% lightness",
       "filter-chip — 26px high, dismiss-on-x, dashed outline for empty add-state"
     ],
     "anti_patterns_in_this_ref": [
       "decorative chart at top adds nothing — would drop in our version"
     ],
     "lift_for_design_system": [
       "chip dimensions",
       "table row height + density",
       "label treatment for column headers"
     ]
   }
   ```

   These are HEURISTIC estimates from the visible screenshot + subagent's knowledge of the source app. Mark estimates clearly with `_estimate` suffix or "(estimated)" qualifiers. Visual refinement (actual color sampling, exact pixel measurement) happens in **Phase 5 synthesis**, not here — keep this phase fast.

5. **Per-screen recommended-pattern paragraph**: synthesize the references into a concrete design direction for this screen. Name layout, key components, density, motion.

Run subagents in parallel (one per surface bucket, 4-6 buckets). Each writes to `data/_atlas-work/results-{LETTER}.json`. Heavy base64 stays in subagent contexts; parent only ever reads the slim JSON outputs.

### Phase 4 — Atlas HTML with Embedded Feedback

Build the atlas as `data/<product-slug>-screen-atlas.html`. **Cream-parchment register** appropriate for partner-facing reading documents. Per-screen sections contain:

- Per-screen header + sub-states + recommended pattern
- Reference grid where each card carries:
  - Image thumbnail (linked from `local_image_path`, per Hard Rule 1)
  - App / flow / analysis (Block A)
  - **Design-system observations panel** (Block B, expandable `<details>`)
  - **Feedback row**:
    ```
    [ skip · ok · lift · hero · avoid ]   (radio group, one click)
    What to lift: __________________________________________ (optional)
    What to skip: __________________________________________ (optional)
    ```
- **Per-screen direction strip** above each grid:
  ```
  Direction for this screen: [ stay course · push more like X · less like Y · pivot ]
  Steer: ____________________________________________________________
  ```
- **Per-surface approach textarea** at top of each surface section:
  ```
  Approach for this surface (2-3 lines): ____________________________
  ```

**Embedded JS (~50 lines, no framework):**

- On any input change, walk the DOM, build a JSON object: `{global_register_steer, buckets: {bucket_id: {approach, sections: {section_id: {direction, steer, refs: {ref_key: {rating, note, but_not}}}}}}}`
- Persist to `localStorage` so refresh/close-tab doesn't lose work
- Sticky bottom bar: `[ Export feedback as JSON ]` button → `navigator.clipboard.writeText(json)` → toast confirmation
- A `[ Reset ]` button (with confirm) clears localStorage

**Accessibility (per ui-design-system rules + Hard Rule 17):**
- Every radio has an `<label>` association
- Every textarea has `<label>` + `aria-describedby` for hints
- 4.5:1 color contrast minimum for ALL text (test before write)
- Keyboard navigable (radio groups via arrow keys, sections via Tab)
- `prefers-reduced-motion` respected
- 16px minimum font size to prevent iOS zoom
- Focus-visible rings on all interactive elements
- `role="radiogroup"` + `aria-label` on rating row
- `tabular-nums` on stats counters
- No `transition: all` — specific properties only

**Output:** single self-contained file at `data/<product-slug>-screen-atlas.html`. Plus a sibling `<product-slug>-screen-atlas.md` with screen list + recommended patterns (terminal-friendly).

### Phase 4.5 — Feedback Gate (HARD GATE)

After Phase 4, **STOP**. Output:
- Path to atlas HTML (open it for the user)
- One-line instructions: "Triage in your own time. Click 'Export feedback as JSON' when done. Paste back to chat."
- DO NOT proceed to synthesis until the user pastes feedback JSON or explicitly says "synthesize without feedback" (rare).

When feedback JSON arrives:
1. Parse it
2. Confirm interpretation back to the user in a short summary: per-surface counts of hero/lift/avoid + the steers verbatim
3. Wait for "go" before Phase 5

Save the parsed feedback to `data/_atlas-work/feedback.json` for the synthesis stage.

### Phase 4.75 — Component Buffet (Path B — Real DOM, NOT Cropped JPEGs)

The buffet is where the user picks per-category component variants (button · input · card · chip · table-row · sidebar-nav · kpi-tile · loading · empty-state · etc.) from candidate sets extracted across the rated references. **All variants MUST render as real HTML/CSS DOM elements, NEVER as cropped screenshots.** Cropping degrades with zoom, framing is imprecise, and the user can't trust a fidelity-blurred image to make a variant-pick.

**Workflow — Path B (vision reconstruction → real HTML/CSS):**

For each component category (one buffet section per category):

1. **Locate candidate variants** across `hero` + `lift` rated references. Aim for 3-6 distinct visual variants per category.

2. **Per-variant extraction** — for each candidate component crop:
   - PIL pixel-sample at center + 4 edges + shadow region → exact hex codes for `bg`, `border`, `text`, `shadow`
   - Vision model measures: padding (top/right/bottom/left to single-pixel precision), border-radius, border-width, font-size, font-weight, shadow spread + offset + opacity
   - Identify font-family approximately (grotesque / serif / mono / display) and map to closest free Google Fonts equivalent
   - Capture visible states if shown in source: hover / focus / disabled / loading / error
   - Save extracted JSON per variant: `data/_atlas-work/buffet/{category}/{ref-id}.json`

3. **Per-variant code generation** — for each extracted JSON, emit inline HTML/CSS that recreates the component as live DOM. Same code shape engineering will use downstream:
   ```html
   <button class="variant-btn" style="
     background: #1a1a1a; color: #fff;
     padding: 10px 18px; border-radius: 8px;
     border: 1px solid #2a2a2a;
     font: 600 13px/1 'Geist', sans-serif;
     box-shadow: 0 1px 2px rgba(0,0,0,0.05), inset 0 1px 0 rgba(255,255,255,0.08);
   ">Run query</button>
   ```
   Each variant card carries: real rendered DOM + variant-id label + source-app attribution + a confidence indicator for vision-estimated values.

4. **Buffet UI per category section:**
   - Title: category name (e.g., "Button") + counter (`5 variants`)
   - Grid of variant cards. Each card:
     - Real rendered component at picker scale (60-100px)
     - Below: `[ pick this ]` radio + variant-id + source-app name
     - "↔ source" toggle (default off) — flips card to show the original cropped screenshot for fidelity verification
     - Inspect-tokens panel (collapsed `<details>`) — shows the extracted JSON
   - Per-category prompt at bottom: "Which one matches your intent? If none, write a steer." (textarea)

5. **Buffet persistence** — same localStorage pattern as Phase 4 atlas. User submits picks as JSON; synthesis (Phase 5) uses those picks as locked component shapes.

**Path A (Playwright DOM scrape) is REJECTED for this stage.** Reasons: most Mobbin sources are auth-gated or mobile apps where DOM scraping fails; browser-install maintenance tail is real engineering cost; incremental fidelity gain over Path B is small for variant-picking (gestalt-level decisions). Add Path A LATER if Path B reconstruction fails on a specific brand.

**Verification before next phase:**
- All variant cards render visible DOM (asset-verification gate per Hard Rule 8)
- Source-toggle flips correctly to source screenshot for fidelity check
- Confidence indicators visible on vision-estimated values

Save buffet picks to `data/_atlas-work/buffet_selections.json` for synthesis.

### Phase 5 — Design-System Synthesis (Visual Deck Only)

Trigger: user says "go" / "synthesize" after the feedback gate.

Read inputs:
- `data/_atlas-work/alignment.json`
- `data/_atlas-work/results-A..F.json` (with observation blocks)
- `data/_atlas-work/feedback.json`

**Optional visual-refinement pass:** for the references rated `hero` or `lift`, optionally fetch the image_url and do an actual color-sampling pass to refine `dominant_palette_estimate` from approximations to exact hex codes. Skip this pass for `skip`/`avoid` rated refs.

Aggregate observations weighted by feedback rating:
- `hero` refs contribute weight 3
- `lift` refs weight 2
- `ok` refs weight 1
- `skip`/`avoid` refs weight 0 (or negative for avoid)

Synthesize a locked design-system deck at `data/<product-slug>-design-system.html`. Sections:

1. **The Two Registers** (cream doc / dark product or whatever the alignment specified)
2. **Color** — 12–20 named tokens with hex + usage matrix + forbidden moves
3. **Typography** — full scale with families/sizes/weights/letter-spacing/line-height + role assignment
4. **Spacing & Density** — 4-based scale + density tokens (compact/standard/comfortable row heights)
5. **Motion** — durations, easings, what's forbidden
6. **Iconography** — stroke weight, sizes, library
7. **Components** (~9 core) — each rendered as inline-SVG specimen WITH explicit text fills (NEVER inherit from global SVG CSS) AND a code-snippet block:
   - JSX + Tailwind class string per component
   - All states (default/hover/active/disabled/focused/error/loading)
8. **Hero Wireframes** (3–5 load-bearing surfaces) — full-page SVG applying the system, with `data-state` toggle so the same SVG can render empty/loading/populated variants
9. **Token Export Block** — copy-pastable code blocks:
   - `tokens.json` (framework-agnostic)
   - `tailwind.config.ts` (theme extension snippet)
   - `theme.css` (CSS variables for runtime)
10. **Mock Data Schemas** — TypeScript types per entity the wireframes reference + a sample `mockData.ts` exemplar
11. **Interaction & Routing Spec** — route map, keyboard shortcuts, focus management rules
12. **Accessibility Checklist** — WCAG-baked rules per component (per `ui-design-system` skill)
13. **Content Spec** — copy register, label conventions, mono identifier naming rule

**Hard contrast guard before write:**

For every SVG specimen on dark backgrounds, walk every `<text>` element and assert it has an EXPLICIT `fill` attribute drawn from product palette tokens (NOT inheriting from global `svg text { fill: var(--ink) }`). If any text element lacks an explicit fill, FAIL the build with a clear error pointing at the offending element.

**Output:** `data/<product-slug>-design-system.html` (Phase 5 ships hero/main page wireframes only — Phase 5.5 below adds the layered interactions on top before audit).

### Phase 5.5 — Layers & Interactions (REQUIRED before audit + handoff)

Phase 5 ships full-page wireframes. Real products also contain **layered interactions** that sit ON TOP of pages — these are equally load-bearing and equally need to be designed before code, NOT improvised in scaffold. Layer specimens are appended to the design-system deck as a new "Layers & Interactions" section.

**Required layer specimens (8 minimum, more if the product needs them):**

1. **Help/Chat Sidebar** — universal "ask about this page" panel that slides in from the right. Spec includes: width, context strip showing what underlies it, conversation thread example, citation chips inline, input affordance, dismiss methods.
2. **Slide-over panel pattern** — generic right-side panel (40-60% viewport width) that hosts secondary surfaces (detail drill-down, inspector, settings drill-down). Spec includes: header treatment, scrim opacity, motion descriptor.
3. **Modal dialog (3 variants)** — standard / destructive / upgrade. Each variant gets its own specimen.
4. **Dropdown menu / filter popover** — anchored below trigger button. Spec includes: search input if applicable, checklist with counts, apply/clear actions.
5. **Command palette (Cmd+K)** — centered modal with global search + grouped results. Spec includes: keyboard shortcuts on results, group structure (Recent · Entities · Actions).
6. **Tooltip + popover** — small hover tooltip + richer hover popover with content preview. Both specimens side by side.
7. **Toast notification stack** — bottom-right anchored, 3 variants (success/error/info). Spec includes: auto-dismiss timing, manual-dismiss X, max-visible.
8. **Right-click context menu** — anchored to cursor. Spec includes: list structure, keyboard-shortcut display, divider groups.

**Each layer specimen must include:**
- Visual render of the layer in OPEN state, on a dimmed/scrim'd mock underlying page (1440×900 canvas matching the wireframe scale)
- **Trigger** — exactly what opens it (button click, keyboard shortcut, right-click, programmatic)
- **Motion descriptor** — enter timing + exit timing + easing function (e.g., `enter: slide-from-right 220ms cubic-bezier(.2,0,0,1) · exit: 160ms ease-in`)
- **Dismiss methods** — Esc / scrim click / explicit close button / auto
- **Focus management** — what receives focus on open, what receives focus on close
- **Z-index assignment** — from the fixed z-index scale (per `ui-design-system` skill: dropdown 50 · sticky 100 · fixed 200 · modalBackdrop 300 · modal 400 · popover 500 · tooltip 600 · toast 700)
- **Keyboard nav notes** — Tab order, arrow keys for menus, Enter/Esc behavior

**Motion vocabulary section** also goes in this phase — a small token table defining reusable motion primitives:
- `motion-enter-from-right` — 220ms `cubic-bezier(.2,0,0,1)` (slide-overs, sidebars)
- `motion-enter-fade` — 180ms `ease-out` (modals, popovers)
- `motion-exit-fast` — 120ms `ease-in` (dismiss)
- `motion-press` — 80ms `ease-in` (button press feedback)

**Hard gate:** Phase 5.5 cannot be skipped. Going directly from full-page wireframes (Phase 5) to scaffold (Phase 6) forces engineering to invent layer treatments in code, which produces inconsistent overlays + animations + dismiss behavior across the app.

**Output:** Append a new section to `data/<product-slug>-design-system.html` titled "Layers & Interactions" with the 8+ layer specimens + motion vocabulary table.

### Phase 5.75 — Pre-Ship Audit Gates (MANDATORY)

Before declaring the design-system deck done, run these three gates IN ORDER. Each must pass before the deck is shown to the user.

**Gate A — Discriminator Audit** (per Hard Rule 6):
Walk EVERY visual primitive across every wireframe + every component specimen. Verify polish parity. Checklist:
- [ ] Buttons + hover/focus/disabled states (layered shadow + inner-highlight inset + hover translateY + focus ring)
- [ ] Cards + hover treatment (layered shadow + translateY)
- [ ] Charts (gradient fills + inner shine + animated reveals)
- [ ] Step badges (done = green gradient + halo + check icon · active = orange gradient + pulse ring · pending = recessed inset shadow)
- [ ] Connector lines between steps (4px gradient bars, NOT 2px solid)
- [ ] Progress bars (gradient fill + inner top shine + animated shimmer sweep + outer brand glow)
- [ ] Connector / brand tiles + logos (framed tile with 1px line + radius + panel bg + layered shadow + inset highlight)
- [ ] Dividers + separators
- [ ] Numbered indicators / pills / chips
- [ ] Icons + their containers
- [ ] Empty states + skeletons (shimmer animation)
- [ ] Form inputs + focus glow ring
- [ ] Status dots + pulse animations on syncing states
- [ ] Background gradients + inset highlights

If ANY primitive is at default-CSS quality (flat solid color, 1px border, no shadow, no hover affordance), fix before proceeding.

**Gate B — Asset Verification** (per Hard Rule 8):
Run silent-failure surface checks:
```bash
# 1. All external image URLs return HTTP 200
for url in $(grep -oE 'https://[^"]+' data/<product>-design-system.html | sort -u); do
  code=$(curl -sI -o /dev/null -w "%{http_code}" "$url")
  echo "$url → $code"
done | grep -v "→ 200"   # any line that isn't 200 is a failure to investigate

# 2. All SVG paths use pure numeric coords (NO % units inside d="...")
grep -E 'd="[^"]*%' data/<product>-design-system.html && echo "FAIL: SVG path with % units"

# 3. Visually open the deck + each standalone wireframe at native size
open data/<product>-design-system.html
for f in data/wireframes/*.html; do open "$f"; done
```

Investigate every non-200 URL. For Simple Icons specifically: some popular brand slugs are TRADEMARK-REMOVED from the public Simple Icons CDN and must use inline SVG or brand-color lettermarks instead. Verify each brand logo URL before shipping.

**Gate B.5 — Multi-Width Layout Check** (per Hard Rule 11):
- [ ] Open every wireframe + layer specimen at native target width (1440 typically) — looks correct
- [ ] Open every wireframe + layer specimen at deck's content-column width (~700-900px) — anything crushed?
- [ ] No text wrapping mid-phrase due to compression
- [ ] No buttons colliding or clipping
- [ ] No 3+ column variant grids using `1fr` (must be `repeat(N, MINPX)`)
- [ ] Modal action buttons centered, not right-aligned
- [ ] If ANY surface fails at narrow width, fix the container to scroll — NEVER shrink the content

**Gate C — Pre-Ship Self-Critical Pass** (per Hard Rule 10):
Re-read the deck + each wireframe AS IF YOU WERE THE USER. Specifically scan for:
- Polish trickle-down failures (any primitive at default quality)
- Wrong genre (tile-cards where a briefing-document was wanted, list where a tree was wanted)
- Generic placeholder content / lorem-style copy where specific product-realistic copy is possible
- Bland palette / muted register that doesn't match the reference companies named in Phase 1
- Cropped JPEG variants in the buffet section (should be real DOM per Phase 4.75)
- Anything that would make the user say "this still doesn't feel premium"

If you would correct it as the user, correct it BEFORE showing them. The iteration-minimization KPI demands this pass.

**Output:** `data/<product-slug>-design-system.html` — only AFTER all three gates pass.

### Phase 6 — Handoff (Hand to Sister Skill)

After Phase 5, output:
1. Path to atlas HTML
2. Path to feedback JSON
3. Path to design-system deck
4. Suggested next: `/scaffold <product>` to turn the locked design-system deck into a working clickable+animated localhost Next.js frontend with mock data
5. Save the alignment + feedback summary to your persistent memory / project notes under the product's domain

The scaffold skill (`skills/scaffold/SKILL.md`) is a SISTER skill that:
- Reads the locked design-system HTML
- Generates the Next.js project (package.json, tailwind.config.ts, theme.css, layout shell, route stubs, components from the JSX snippets, mock data from the schemas)
- Runs `npm install`
- Boots `npm run dev` with a process monitor attached
- Smoke-tests each route (curl + grep rendered HTML against expected wireframe markers)
- Runs a contrast pass on each rendered page
- Iterates fixes until green
- Hands back a verified localhost URL

`/scaffold` is invoked separately AFTER the user reviews the design-system deck.

## Common Failure Modes

- **Skipping Phase 2 confirmation** — burning Mobbin queries on a wrong screen list wastes the run. Always confirm sitemap.
- **Generic per-screen analysis** — if the analysis would apply to any product, rewrite it. Anchor every analysis to *this* product's *this* screen.
- **Hero references that miss the register** — filter by register fit during curation, not after.
- **Skipping the feedback gate** — DO NOT auto-synthesize after atlas. Wait for user.
- **Calling Mobbin tools that don't exist** — discover names via ToolSearch before writing code.
- **SVG text inheriting global fill on dark specimens** — always explicit `fill="..."` from product palette.

## Output Quality Bar

A finished atlas should let the user triage in 15-30 minutes (rate refs, write a few steers, export JSON). The synthesis after feedback should produce a design-system deck rich enough that the sister `/scaffold` skill can generate a working Next.js project from it WITHOUT requiring further design decisions. Every component spec, every token, every wireframe state must be unambiguous code-input-quality.
