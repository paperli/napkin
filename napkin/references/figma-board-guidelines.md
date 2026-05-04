# Figma Board Guidelines

Conventions for how Napkin renders a Napkin Flow IR onto a Figma file. These rules govern structure, naming, and visual style of the output. The Plugin-API JS that does the drawing is in `figma-rendering-recipes.md`.

Phase-2a scope: first render and whole-board re-render. Targeted iteration is Phase-2b.

---

## Tool surface

Napkin uses the **official Figma MCP** (`mcp__claude_ai_Figma__*`). The relevant tools:

| Purpose | Tool |
|---|---|
| Get authenticated user / `planKey` | `whoami` |
| Create a new design file | `create_new_file` |
| Run Plugin-API JS in a file (the everything tool) | `use_figma` |
| Verify result — structure | `get_metadata` |
| Verify result — visual | `get_screenshot` |

Napkin's writes route almost entirely through `use_figma`. The other write tools (`upload_assets`, `generate_diagram`, `add_code_connect_map`, `send_code_connect_mappings`) are not used in Phase 2a.

---

## File and page structure

Napkin renders into a single Figma **design** file (not FigJam). Inside, Napkin uses Figma **pages** as the top-level grouping:

```
<Figma file>
  00 Overview
    - Flow summary text frame
    - Assumptions text frame (if any)
    - Open questions text frame (if any)
  01 Main Flow
    - Screen frames (one per screen, laid out left-to-right by flow order)
    - Flow arrows (lines + arrowhead triangles between screens)
  02 Optional Branches    (created only if the IR has any flows[].isOptional === true)
    - Branch destination screens
    - Arrows from main flow into branches
  03 Notes                (created only if assumptions or openQuestions exist)
    - UX notes
    - Ambiguities
    - Suggested next steps
```

Page names use the `00 Overview` form (two-digit prefix + space + title) so they sort lexicographically in the Figma sidebar.

---

## File ownership

**Napkin does not own the file. The user does.** Napkin owns specific pages and frames within it, marked by the `[napkin:...]` prefix in their names. Anything in the file *not* marked is not Napkin's to modify, ever — including:

- pages the user added
- frames the user drew
- comments
- version history
- file-level settings (name, library subscriptions, etc.)

This boundary is what makes the official MCP safe to use against a real user file.

---

## Naming conventions

Naming is the bridge between the IR and Figma. After a render, the agent should be able to find any IR-described node by name alone.

| Object | Name format | Example |
|---|---|---|
| Page | `<order> <Title>` | `01 Main Flow` |
| Screen frame | `[napkin:screen:<id>] <Name>` | `[napkin:screen:signup] Signup` |
| Element node | `[napkin:el:<id>] <type>: <label>` | `[napkin:el:signup_email] text_field: Email` |
| Flow arrow shaft | `[napkin:edge:<id>] <fromId> → <toId>` | `[napkin:edge:e1] landing → signup` |
| Flow arrow head | `[napkin:edge:<id>:head]` | `[napkin:edge:e1:head]` |
| Overview text frame | `[napkin:overview:<kind>]` | `[napkin:overview:summary]` |

The `[napkin:...]` prefix is **required** on every node Napkin creates. It is what makes catch-up and Phase-2b targeted iteration work.

---

## Frame sizes by ratio class

Frame dimensions are picked from a **standard-size lookup keyed by the detected `frameRatio`**, not by `viewport` alone. `viewport` is the semantic label (`mobile`, `tablet`, `desktop`, `tv`); `frameRatio` is the rendering hint that determines orientation and aspect.

| Ratio class | Frame (W × H) | Typical use |
|---|---|---|
| `9:19.5` portrait | 390 × 844 | modern phone, portrait |
| `9:16` portrait | 375 × 667 | older phone, portrait |
| `3:4` portrait | 768 × 1024 | tablet, portrait |
| `4:3` landscape | 1024 × 768 | tablet, landscape |
| `16:10` landscape | 1440 × 900 | desktop |
| `16:9` landscape | 1440 × 810 | desktop / lean-back |
| `21:9` ultrawide | 2560 × 1080 | desktop, wide |
| `18:9` landscape | 812 × 375 | rotated phone, landscape |
| `1:1` square | 600 × 600 | ambiguous; ask in 1b before relying on this |

If `frameRatio` is absent (legacy IR or ambiguous detection), the renderer falls back to viewport defaults: `mobile` → 375 × 812, `tablet` → 768 × 1024, `desktop` → 1440 × 900, `tv` → 1920 × 1080.

Standard sizes are not pixel-perfect; the goal is matching orientation and rough aspect, not 1:1 reproduction of what the user drew. Napkin does not resize frames to fit content — content overflow is acceptable for a low-fi wireframe; clarity of the frame box matters more.

---

## Canvas layout

On the `01 Main Flow` page, screens are laid out **preserving the relative positions the user drew**. Each screen carries a normalized `sketchPosition` (0..1 inside its source image); the renderer scales those positions onto a per-source canvas region so a screen drawn top-right in the sketch lands top-right on the Figma board, and screens drawn vertically stacked stay vertically stacked.

- **Group by source sketch.** Screens from the same source image share one canvas region.
- **Stack groups vertically.** Each source becomes its own region, stacked top-to-bottom with a 400 px gap between regions.
- **Inter-frame spacing minimum**: 200 px gutter; the per-source region is sized big enough to keep relative positions readable.
- **Fallback.** If `sketchPosition` is absent on any screen (legacy IR or ambiguous detection), the renderer falls back to linear left-to-right with row wrap at 4 — the previous Phase-2a default.
- **Branches** on `02 Optional Branches` follow the same rules within their page. Cross-page arrows are Phase-2b.
- **Overlap.** If two screens are drawn at near-identical positions in the sketch, they overlap on the Figma board. Phase-2a accepts this — the user's choice reads as deliberate (e.g. modal over parent). Phase-2b can add nudging.

Phase-2a layout algorithm in one line: scale `sketchPosition` onto a per-source region; stack regions vertically; fall back to grid when positions are missing. Smarter graph layout is Phase-2b+.

---

## Element layout within frames

The same position-preserving rule applies one level down: each `NapkinElement` carries optional `sketchPosition` (0..1 centroid within its parent) and `sketchSize` (0..1 fraction of parent's width/height). The renderer scales these onto the actual screen frame so elements land roughly where they were drawn, at roughly the size they were drawn.

- **Reference frame is the parent.** Top-level elements normalize to the screen frame; nested elements (a button inside a card) normalize to the immediate parent container, not the screen. This keeps the math compositional.
- **Loose precision.** Two-decimal rounding is enough; this is an art-directorial hint, not pixel data.
- **Size override.** If `sketchSize` is present, the renderer overrides the archetype default — so a button drawn full-width comes out wide instead of the archetype's 120 px default.
- **Fallback.** If `sketchPosition` is missing, the renderer stacks elements vertically at the left margin (24 px x, 56 px row height). That's a degraded layout; Step 2 should capture positions so the fallback rarely fires.
- **Overlap.** Same Phase-2a stance as screens — overlap is treated as deliberate (modal over content, dropdown over field).

---

## Low-fi visual style

Strict rules — no exceptions in Phase 2a:

| Element | Rule |
|---|---|
| Background | White (`#FFFFFF`) |
| Frame stroke (chrome containers) | 1 px, light gray (`#E5E5E5`) |
| Frame stroke (primary content) | 1 px, dark gray (`#333333`) |
| Text color | Dark gray (`#333333`); secondary text `#777777` |
| Typeface | `Inter` (Figma default) — single typeface across the entire board |
| Type scale | 24 px (heading), 16 px (h2), 14 px (body), 12 px (caption) |
| Fills | Allowed only for buttons (light gray `#F5F5F5`) and disabled states (`#FAFAFA`); everything else is stroke-only |
| Corner radius | 4 px on inputs and buttons; 0 elsewhere |
| Shadows | None |
| Brand colors / logos / imagery | None |
| Icons | Optional, drawn as 16×16 vector glyphs in `#333333`. Skip if unclear in the sketch |

These rules are deliberately rigid. Napkin's purpose is to keep stakeholder discussion on flow and structure, not visuals.

---

## Flow arrows

Design files (unlike FigJam) have no native connector primitive. Napkin draws arrows as primitives:

- **Shaft**: a line, 1 px stroke, dark gray (`#333333`).
- **Arrowhead**: a small filled triangle (12 px long × 8 px wide) at the destination.
- **Label** (if `edge.label` exists): centered above the shaft, 12 px text, `#333333`.
- **Optional edges** (`isOptional === true`): dashed shaft (4-2 dash pattern), label suffixed with `(optional)`.

Phase-2a draws arrows as **straight lines from source-frame center to destination-frame center**. Curved/bent arrows and obstacle avoidance are Phase-2b+.

---

## Render batching

`use_figma` has a 50,000-char limit on the `code` parameter. To stay under it safely:

- **One `use_figma` call per page** is the default.
- If a single page's generated JS would exceed ~30,000 chars, split by **screen group** (5 screens per call).
- Each call's `description` should be a one-line summary (e.g., `"Render 01 Main Flow: 5 screens, 4 arrows"`). This text appears in Figma's version history.

Always load fonts in every call (`figma.loadFontAsync`) — font state is per-call, not per-file.

---

## Verify loop

After all writes complete:

1. `get_metadata` on the file root, confirm expected pages exist with the `[napkin:...]` markers.
2. `get_screenshot` on the `01 Main Flow` page (using the `pageId` Napkin captured) — gives the user a preview link.

Verification is **not transactional**. Phase 2a does not roll back partial renders. If verification surfaces missing pages or frames, tell the user what landed and what didn't, and offer to retry the failing batch.

---

## What the user sees on success

```
Rendered to: https://www.figma.com/file/<fileKey>

Pages:
- 00 Overview
- 01 Main Flow (5 screens, 4 arrows)
- 03 Notes (2 assumptions, 1 open question)

Preview: <screenshot URL>
```

Brief, scannable, file URL up top so the user can click through.

---

## Re-render and iteration (Phase 2a behavior)

Phase 2a supports two render modes:

| Mode | When | Behavior |
|---|---|---|
| **First render** | `figmaBoard.fileKey` is absent in the IR | Resolve fileKey (user URL or `create_new_file`), draw everything, persist `figmaBoard` metadata |
| **Whole-board re-render** | User explicitly asks ("rebuild the Figma board") | **Destructive**: delete Napkin-owned pages, recreate from current IR. Loses user edits inside Napkin-owned frames. Always warn and require confirmation before running |

**Targeted iteration** ("change just screen 3 to a modal") is **Phase-2b**. In Phase 2a, if the user requests a targeted change, Napkin updates the IR but tells the user the Figma board needs either a manual edit in Figma or a whole-board re-render until 2b lands.

---

## Catch-up rendering

If the user fixes their MCP setup or starts a new session and says something like "catch up — render the latest IR":

1. Load `.napkin/flow.json`.
2. **Skip interpretation entirely.** The IR is already approved.
3. Run the rendering pipeline.
4. If `figmaBoard.fileKey` is populated, target that file. If not, prompt the user for a URL (or accept "create a new file").

Catch-up is structurally identical to a first render or whole-board re-render — the only difference is Step 1–3 of the Skill workflow are skipped.
