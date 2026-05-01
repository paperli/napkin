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

Also collect or assume:

- **Target surface** (default: `responsive_web`). Other valid values: `mobile_app`, `desktop_app`, `tv`. Phase-1 implementation focus is responsive web; later phases add the rest.
- **Product context** — a one-line description of what the flow is for.
- **Target viewport(s)** when ambiguous (desktop, tablet, mobile).

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

## Step 5 — Render to Figma

Detect whether **Figma MCP** is connected by checking the available tool list for Figma MCP tools (e.g. tools whose names start with `mcp__figma` or similar Figma-MCP-provided names).

### If Figma MCP is available

In Phase-1, this skill ships without Figma rendering. Tell the user the IR is ready and that Figma rendering will be available in Phase-2 (`references/figma-board-guidelines.md` ships then). Do not attempt to render in Phase-1.

### If Figma MCP is not available

Tell the user clearly:

- the IR has been saved to `.napkin/flow.json` and `.napkin/flow.md`
- Figma rendering was skipped because Figma MCP is not connected
- they can fix MCP setup and ask Napkin to **catch up** later

### Catch-up rendering (later phases)

If the user runs Napkin again and says something like:

```
Catch up — render the latest IR to Figma now.
```

…then:

1. **Skip interpretation entirely.** The IR is already approved.
2. Load `.napkin/flow.json`.
3. Render the full board to Figma in one pass.
4. Store any board metadata Figma MCP returns (board id, frame ids) back into `.napkin/flow.json` so future iterative diffs can target the correct frames.

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
