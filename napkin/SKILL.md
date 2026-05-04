---
name: napkin
description: Use this skill when the user wants to convert rough UX sketches, paper sketches, whiteboard photos, screenshots, or early user-flow drawings into structured low-fidelity Figma wireframe boards. Napkin interprets sketches through a UX and design-system mindset, asks clarifying questions when needed, creates a Napkin Flow IR, maps rough UI marks to semantic components, and uses Figma MCP tools to draw editable screens, flow arrows, annotations, assumptions, and shareable user-flow documentation in Figma.
---

# Napkin

Napkin transcribes rough product-flow sketches into a structured low-fidelity Figma wireframe board.

Napkin's value is **interpreting design intent**, not tracing pixels. Output is intentionally style-less and low-fidelity so conversations stay focused on flow and structure, not visuals.

---

## When to use this skill

Trigger when the user provides — or refers to — one or more of:

- paper sketches photographed or scanned
- whiteboard photos
- screenshots of rough mockups
- annotated drawings of a user flow

…and asks for a wireframe, flow diagram, Figma board, or "clean version" of those sketches.

If the user only wants to discuss UX without producing a board or IR, do **not** invoke this skill.

---

## Workflow

```
1. Intake     → collect sketches + context
2. Interpret  → identify screens, elements, flows
3. Clarify    → only when needed
4. Generate   → produce Napkin Flow IR, persist to .napkin/
5. Render     → call Figma MCP to draw the board (if available)
6. Iterate    → diff-based revisions on subsequent runs
```

The brain is this Skill. Figma MCP is the drawing hand. All interpretation, normalization, and clarification happens *before* any MCP call.

---

## Step 1 — Intake

Napkin accepts sketches in **two ways**:

- **Inline images** attached to the user's message. Read them via the `Read` tool against the path Claude Code surfaces for the attachment.
- **Local file paths** the user passes in the prompt (absolute or project-relative).

Both are first-class. If the user references a sketch but no image is attached and no path is given, prompt them to drop images at:

```
<project-root>/.napkin/sketches/
```

…and re-run with the path. Do not guess.

Also collect:

- **Product context** — a one-line description of what the flow is for.
- **Target surface** — `responsive_web`, `mobile_app`, `desktop_app`, or `tv`. If the user didn't say, infer it (1a) and confirm (1b). Phase-1 implementation focus is responsive web; later phases add the rest.
- **Per-screen viewport** — `desktop`, `tablet`, `mobile`, or `tv`. Inferred per sketch (1a).

### 1a. Detect viewport from drawn sketch frames

Infer viewport from the **aspect ratio of the drawn sketch frame** — not the image's aspect ratio. A phone photo of a desktop sketch is portrait at the photo level but the drawn frame is landscape. A whiteboard photo of three mobile screens is landscape at the photo level but each drawn frame is tall and portrait. The signal is the rectangle the user drew, not the photo crop.

For each sketch:

1. Find the rectangle(s) the user drew as screen frames. If a sketch contains multiple screen frames, treat each independently.
2. Estimate the frame's width:height ratio.
3. Combine with content cues (sidebars, bottom tab bars, mobile status-bar mockups, focus highlights, rails, browser chrome).
4. Map to a viewport guess using the table below.

| Drawn frame ratio | Default guess | Override cues |
|---|---|---|
| ~9:16 – 9:19.5 (tall portrait) | `mobile` | sidebar + dense table → `desktop` (narrow window) |
| ~3:4 (portrait) | `tablet` | bottom tab bar → `mobile`; sidebar + dense table → `desktop` |
| ~4:3 (landscape) | `tablet` | dense table + multi-pane → `desktop`; rails + focus highlight → `tv` |
| ~16:9 / 16:10 (wide) | `desktop` | rails + focus highlight + lean-back framing → `tv` |
| ~21:9 (ultra-wide) | `desktop` | rails + focus → `tv` |
| ~1:1 (square) | ambiguous — ask | — |

Then derive `targetSurface`:

- All sketches resolve to `mobile` → propose `mobile_app`.
- All sketches resolve to `desktop` → propose `desktop_app` if dense desktop chrome (menus, sidebars, multi-pane), else `responsive_web` (especially with browser chrome).
- Mixed `mobile` + `desktop` → propose `responsive_web` with per-screen `viewport` set per sketch.
- Any sketch resolves to `tv` → propose `tv`.

### 1b. Confirm the guess

Surface the guess as a single confirmation turn before generating the IR:

```
Detected sketch frames:
- sketch_01.png — ~9:18  → mobile
- sketch_02.png — ~16:9  → desktop
- sketch_03.png — ~16:9  → desktop

Best guess: responsive_web (1 mobile screen + 2 desktop screens).

Confirm, or correct (e.g. "all mobile_app", or "sketch_02 is tablet").
```

Resolution:

- If the user already named the surface in their prompt **and** it matches the dominant detected ratio, skip this confirmation and record per-screen viewports as `assumptions`.
- If the user named a surface that **conflicts** with what every sketch shows (e.g. "mobile_app" but all sketches are 16:9), surface the conflict and ask — don't silently pick either side.
- If the user corrects a single sketch, update only that screen's `viewport`. Don't propagate to other screens.
- Record rationale on each detection-based `assumption` so the user can audit it later, e.g. `"Inferred 'mobile' for screen 'signup' from ~9:18 drawn frame ratio + bottom tab bar."`.

---

## Step 2 — Interpret

Read `references/ux-interpretation-principles.md` before reasoning about the sketches. It explains *how* to look at a sketch (intent over marks, hierarchy, repeated patterns, missing states).

Read `references/component-taxonomy.md` to map rough marks to semantic components (a top rectangle is usually a header, three repeated cards are usually a card list, etc.).

The output of this step is a **draft** Napkin Flow IR — held in your reasoning, not yet written to disk.

---

## Step 3 — Clarify

Read `references/clarification-questions.md` for triggers and patterns.

Ask **only** when:

- screen boundaries are unclear
- arrows are ambiguous (navigation? data dependency?)
- target platform is unspecified and signals conflict
- multiple plausible interpretations exist
- important labels are unreadable
- the flow is missing a start or end state
- the user's stated intent conflicts with what the sketch shows

If you have enough context, **proceed and mark assumptions** rather than asking. The bar for an open question is "this materially changes the output." Anything below that bar goes into `assumptions` on the IR.

Bad behavior:

```
Please explain every screen before I continue.
```

Good behavior:

```
I'll assume the top-left screen is the start state and the bottom-right is the success state.
Two things I need from you before I render:
1. Is the team-invite screen optional or required?
2. Desktop only, or desktop + mobile?
```

---

## Step 4 — Generate IR

Read `references/napkin-flow-ir-schema.md` for the full schema (v0.2) and validation rules.

Persist the IR to:

```
<project-root>/.napkin/
  flow.json    # canonical IR (machine-readable)
  flow.md      # human-readable summary of the same flow
  sketches/    # original sketch images, copied here if the user provided paths elsewhere
  history/     # snapshots of prior IR versions (on iteration)
```

Use the `Write` tool. Create `.napkin/` if it does not exist. Do not put the IR anywhere else — downstream catch-up and iteration runs depend on this path.

`flow.md` template is in `references/napkin-flow-ir-schema.md`.

---

## Step 5 — Render to Figma (Phase 2a)

Read `references/figma-board-guidelines.md` for board structure, naming, and visual rules.
Read `references/figma-rendering-recipes.md` for Plugin-API JS patterns.

### 5a. Detect Figma MCP

Check the available tool list for `mcp__claude_ai_Figma__use_figma`. If absent, **enter IR-only mode**: tell the user the IR is saved at `.napkin/flow.json` and `.napkin/flow.md`, that Figma rendering was skipped because Figma MCP is not connected, and that they can fix MCP and ask Napkin to **catch up** later.

### 5b. Resolve fileKey

**On the first render** (no `figmaBoard.fileKey` in the IR yet):

- **Default**: ask the user for a Figma file URL. Extract the fileKey from `https://www.figma.com/file/<fileKey>/...` or `https://www.figma.com/design/<fileKey>/...`.
- **If the user explicitly says "create a new file"**: call `mcp__claude_ai_Figma__whoami` to get `planKey`, then call `mcp__claude_ai_Figma__create_new_file` with `editorType: "design"` and `fileName` set from the IR's `projectName`. Capture the returned `fileKey`.

Persist `fileKey` to `.napkin/flow.json:figmaBoard.fileKey`.

**On subsequent renders**: read `figmaBoard.fileKey` from `.napkin/flow.json`. Do not ask again.

### 5c. Plan the render

Per `figma-board-guidelines.md`, plan up to four pages:

- `00 Overview` — always
- `01 Main Flow` — always
- `02 Optional Branches` — only if any `flows[].isOptional === true`
- `03 Notes` — only if `assumptions.length > 0` or `openQuestions.length > 0`

Decide batching:

- **One `use_figma` call per page** is the default.
- If a single page's generated JS would exceed ~30,000 chars, split by **screen group** (5 screens per call).

### 5d. Generate Plugin-API JS

For each batch, generate JS that:

1. Loads required fonts via `figma.loadFontAsync`.
2. Defines the helpers and constants from `figma-rendering-recipes.md`.
3. Creates or finds the target page (via `ensurePage`).
4. Creates frames, elements, and arrows per the recipes, applying the `[napkin:...]` naming conventions to every node.
5. Returns `{ pageId, frameIds, lastRenderedAt }`.

### 5e. Execute the writes

Call `mcp__claude_ai_Figma__use_figma` with:

- `fileKey`: from `figmaBoard.fileKey`.
- `code`: the generated JS.
- `description`: a short summary of the batch (e.g., `"Render 01 Main Flow: 5 screens, 4 arrows"`). This appears in Figma's version history.

Capture the return value from each call.

### 5f. Persist Figma metadata

After all batches succeed, write back to `.napkin/flow.json:figmaBoard`:

- `fileKey` — already set
- `pageId` — Main Flow page id
- `frameIds` — merged map `{ screenId: figmaFrameId }` from all batches
- `lastRenderedAt` — ISO timestamp

This metadata is what makes Phase-2b targeted iteration and cross-session catch-up work.

### 5g. Verify

Recommended after each render:

1. Call `mcp__claude_ai_Figma__get_metadata` on the file root to confirm expected pages exist.
2. Call `mcp__claude_ai_Figma__get_screenshot` on the Main Flow page to give the user a preview.

If verification surfaces missing pages or frames, tell the user what landed and what didn't. Phase 2a does not roll back partial renders.

### 5h. Report

```
Rendered to: https://www.figma.com/file/<fileKey>

Pages:
- 00 Overview
- 01 Main Flow (<n> screens, <m> arrows)
- 02 Optional Branches (<n> branches)             (if rendered)
- 03 Notes (<n> assumptions, <m> open questions)  (if rendered)

Preview: <screenshot URL>
```

### Re-render and iteration (Phase 2a)

Phase 2a supports two render modes only:

- **First render** — `figmaBoard.fileKey` absent in the IR. Run the full pipeline above.
- **Whole-board re-render** — user explicitly asks ("rebuild the Figma board"). **Destructive**: deletes Napkin-owned pages and recreates them, losing user edits inside Napkin-owned frames. Always warn and require confirmation before running.

**Targeted iteration** ("change just screen 3 to a modal") is **Phase 2b**. In Phase 2a, if the user requests a targeted change:

1. Update the IR to reflect the change and persist.
2. Tell the user the IR is updated, and the Figma board does not yet support targeted patching — they can either edit the affected frame manually in Figma, or request a whole-board re-render, or wait for Phase 2b.

### Catch-up rendering

If the user fixes their MCP setup or starts a new session and says something like "catch up — render the latest IR":

1. Load `.napkin/flow.json`.
2. **Skip interpretation entirely.** The IR is already approved.
3. Run Step 5 from 5a.
4. If `figmaBoard.fileKey` is populated, target that file. If not, prompt for a URL (or accept "create a new file").

---

## Iteration model

When the user asks for revisions on an existing flow ("make screen 3 a modal", "add an error state after signup"):

1. Load `.napkin/flow.json`.
2. Compute a **targeted patch** — modify only the affected screens, elements, or edges.
3. Snapshot the prior IR to `.napkin/history/<ISO-timestamp>.json` before overwriting.
4. Save the updated IR to `.napkin/flow.json` and refresh `.napkin/flow.md`.
5. If Figma MCP is connected, re-render only the affected frames — not the whole board.

Do not regenerate everything for a small change. Do not destroy user edits made directly in Figma without warning the user first.

---

## What Napkin must not do

- Apply colors beyond a small grayscale ramp.
- Apply brand styling, brand fonts, logos, or product imagery.
- Generate high-fidelity UI or pixel-perfect mockups.
- Produce flattened images in Figma — frames must remain editable.
- Trace the sketch literally — interpret intent, not marks.
- Over-ask. If you have enough context, proceed and document assumptions.
- Ingest the user's existing design system as a styling source (Phase-1 ignores it; later phases will accept *headless-style* specs only).
- Invent business logic. If a transition is unclear, mark it as an open question.

---

## Style-less by default

Output is grayscale, unbranded, low-fidelity. The reason: Napkin's job is to keep stakeholder discussion on **flow and structure**, not on visual details. Branded or styled output drags the conversation into the wrong layer at the wrong stage.
