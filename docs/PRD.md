# Napkin PRD

**Product name:** Napkin  
**Code name:** Project Napkin  
**Version:** 0.3  
**Primary surface:** Claude Code Skill  
**Execution bridge:** Figma MCP (official — `mcp__claude_ai_Figma__*`)  
**Primary output:** Shareable low-fidelity Figma user-flow wireframe board  
**Secondary output:** Persisted `Napkin Flow IR` file (works without Figma MCP)

---

## 1. Product Overview

Napkin is a Claude Code Skill that converts rough user-flow sketches into structured, low-fidelity Figma wireframe boards.

Users provide photos, scans, screenshots, or whiteboard images of early product-flow sketches. Napkin interprets the sketches through a UX and design-system lens, asks clarifying questions when needed, maps rough UI marks to semantic interface structures, and uses Figma MCP to create editable screens, flow arrows, annotations, and shareable wireframe documentation in Figma.

Napkin is not intended to trace sketches literally or generate high-fidelity UI. Its core value is to understand rough design intent and transcribe it into a clean, legible product-flow artifact.

---

## 2. One-Liner

**Napkin turns rough product-flow sketches into clean, shareable Figma wireframe boards.**

---

## 3. Positioning

Napkin is an AI design-transcription workflow for early-stage product design.

It sits between messy ideation and formal wireframing:

```text
Sketches + product context
→ Claude Code Skill asks clarifying UX questions
→ Skill interprets screens, flows, and UI intent
→ Skill creates structured Napkin Flow IR
→ Skill calls Figma MCP
→ Figma board contains clean low-fidelity wireframes and flow graph
```

Napkin should feel like a design partner that can read rough sketches, understand product intent, and prepare a structured Figma document that teams can critique, share, and iterate on.

---

## 4. Problem Statement

Early product ideas are often captured as rough sketches on paper, whiteboards, tablets, or screenshots. These sketches are fast for thinking but slow to convert into clean, shareable design artifacts.

Designers, PMs, and founders often need to manually:

- recreate screens in Figma
- clean up rough layouts
- rename screens
- redraw arrows
- normalize inconsistent UI elements
- add notes and assumptions
- make the flow understandable to stakeholders

Existing AI image-generation tools can produce polished-looking mockups, but they often fail at product semantics. They produce visual artifacts instead of editable, structured, low-fidelity UX documentation.

Napkin solves this by turning rough sketches into semantic user-flow wireframes on Figma.

---

## 5. Product Goal

Create a Claude Code Skill that can:

1. Understand rough user-flow sketches.
2. Ask UX-aware clarification questions.
3. Convert sketches into a structured flow representation.
4. Normalize screens using design-system principles.
5. Use Figma MCP to draw clean, editable, shareable wireframe boards.

---

## 6. Core Thesis

Napkin should not simply trace sketches.

Napkin should **transcribe design intent**.

That means it should recognize:

- screen boundaries
- user journeys
- navigation paths
- UI hierarchy
- repeated design patterns
- component intent
- ambiguous areas
- missing states
- responsive layout implications
- design-system structure

The output should be a legible design document in Figma, not a polished UI mockup.

---

## 7. Target Users

### Primary Users

- Product designers
- UX designers
- Founders
- PMs
- Prototypers
- Design technologists

### Secondary Users

- Engineers creating product concepts
- UX researchers
- Startup teams
- AI-assisted product teams

---

## 8. Primary Use Cases

### Use Case 1: Paper Sketch to Figma Flow

A user uploads photos of paper sketches. Napkin identifies screens, components, and arrows, then creates a clean Figma flow board.

### Use Case 2: Whiteboard Workshop Cleanup

A user provides a photo of a whiteboard after a brainstorm. Napkin turns it into a structured flow diagram and wireframe set.

### Use Case 3: Product Concept to Wireframe Board

A user provides sketches plus a text prompt such as:

```text
This is a SaaS onboarding flow for a team workspace product.
Make this a responsive web flow.
```

Napkin asks clarifying questions, then creates a coherent Figma artifact.

### Use Case 4: Design-System-Aware Transcription

A user says:

```text
Use headless UI / Radix-like structure.
Keep it low fidelity.
Make the screens consistent.
```

Napkin maps rough sketch elements into semantic components such as nav bars, tabs, cards, forms, dialogs, tables, and empty states.

### Use Case 5: Shareable UX Document

A user wants a Figma board that can be shared with stakeholders, not a final design. Napkin creates:

- screen map
- low-fi wireframes
- flow arrows
- assumptions
- open questions
- UX notes

---

## 9. MVP Scope

### MVP Platform Focus

Start with **Responsive Web Design**.

Rationale:

- It is broad and useful.
- It maps well to design-system structures.
- It supports desktop and mobile thinking.
- It avoids mobile-native and TV-specific complexity at launch.
- It is easier to normalize into low-fi wireframes.

### MVP Input

Napkin should accept sketches in **two ways**:

1. **Inline images pasted into the conversation** (the default for many users).
2. **Local file paths** to one or more sketch images, screenshots, scans, or whiteboard photos.

Both input modes must be supported. When inputs are ambiguous (for example, a screenshot was attached but Napkin cannot resolve it as a readable image, or the user references a sketch but provides no image), Napkin should prompt the user to drop the image into a conventional project location and provide its path:

```text
~/Documents/Projects/<your-project>/.napkin/sketches/
```

Napkin should accept all of:

- inline images attached to the message
- absolute or project-relative paths to sketch images
- screenshots
- scanned paper images
- whiteboard photos
- optional written context
- optional product type
- target viewport — auto-detected per sketch from the drawn frame's aspect ratio at intake (see SKILL §1a); user may confirm or override per sketch
- target surface — Phase-2a always renders `responsive_web`; if the user names `mobile_app`, `desktop_app`, or `tv` it is recorded on the IR but rendering is deferred to Phase-4/5
- optional design-system reference *(future / low priority — see §15)*

Example user prompt:

```text
Use Napkin to convert these sketches into a responsive web onboarding flow in Figma.
Ask me questions if anything is unclear.
```

### MVP Output

A Figma board containing:

- named flow section
- screen frames
- low-fidelity wireframes
- flow arrows
- screen titles
- UX annotations
- assumptions
- unresolved questions
- optional component legend

### MVP Supported UI Patterns

Responsive web:

- landing page
- signup/login
- onboarding
- dashboard
- settings
- list/detail
- form flow
- modal flow
- checkout-like step flow
- empty/error/loading states

---

## 10. Non-Goals

Napkin should not try to:

- generate high-fidelity visuals
- apply final branding
- create production-ready React code
- replace designer judgment
- infer complex business logic without asking
- support all devices in MVP
- create pixel-perfect recreations of sketches
- turn every rectangle into a generic box
- output flattened images in Figma

The output should remain intentionally low-fidelity and editable.

---

## 11. Recommended Product Architecture

### Primary Architecture

```text
Claude Code Skill: Napkin
        ↓
Sketch intake + context gathering
        ↓
UX interpretation engine
        ↓
Design-system normalization
        ↓
Napkin Flow IR
        ↓
Figma rendering plan
        ↓
Figma MCP
        ↓
Shareable Figma board
```

### Key Principle

Figma MCP should be treated as Napkin's **drawing hand**, not its brain.

Napkin's brain is the Skill:

- UX interpretation
- Design-system mapping
- Information architecture
- Clarification
- Screen decomposition
- Flow graph reasoning
- Wireframe planning

Figma MCP's job is:

- create Figma frames
- create text layers
- create rectangles/components
- create flow arrows
- arrange screens
- apply naming conventions
- add annotations

---

## 12. Why Claude Code Skill First

### Broader Reach Than a Figma Plugin

A Figma Plugin only works once the user is already inside Figma and has installed the plugin. A Claude Code Skill can sit closer to the thinking process:

- user has sketches in a folder
- user has notes in Markdown
- user has product requirements
- user has screenshots
- user has existing design-system docs
- user wants to iterate conversationally before committing to Figma

### Better for Ambiguity

Sketches are ambiguous. Napkin needs a conversational clarification loop:

- Are these separate screens or states of one screen?
- Is this arrow a navigation path or data dependency?
- Should this be mobile-first or desktop-first?
- Should Napkin preserve the rough layout or normalize it?
- Is this modal part of checkout or account setup?
- Should the output be a user journey board, wireframe board, or product spec board?

### Better for Design-System Reasoning

Napkin needs reusable design intelligence. A Skill can include reference files such as:

```text
napkin/
  SKILL.md
  references/
    ux-interpretation-principles.md
    component-taxonomy.md
    responsive-web-patterns.md
    mobile-patterns.md
    tv-navigation-patterns.md
    figma-rendering-guidelines.md
    clarification-questions.md
    output-spec-schema.md
  scripts/
    validate_flow_ir.ts
    normalize_component_tree.ts
    generate_figma_plan.ts
```

---

## 13. Core User Experience

### Ideal Interaction

```text
User:
Use Napkin to turn these sketches into a Figma flow.
Target: responsive web SaaS onboarding.

Napkin:
I see what looks like 5 screens: landing, signup, invite team, setup workspace, dashboard.
A few things are ambiguous:
1. Is the “team” screen required or optional?
2. Should the dashboard be an end state or part of the onboarding?
3. Should I generate desktop-only or desktop + mobile frames?

User:
Team invite is optional. Dashboard is the end state. Desktop only for now.

Napkin:
Great. I’ll create a low-fidelity Figma board with:
- 5 screen frames
- one optional branch
- Radix-like component structure
- flow arrows
- assumptions panel
```

Then Napkin calls Figma MCP to draw the board.

---

## 14. Skill Behavior Requirements

### Napkin Must Ask Questions When Needed

Napkin should ask clarifying questions before rendering if:

- screen boundaries are unclear
- arrows are ambiguous
- target platform is unspecified
- multiple interpretations are plausible
- important labels are unreadable
- user intent conflicts with sketch structure
- flow has missing start/end states
- interaction model is unclear

### Napkin Should Not Over-Ask

If enough context exists, Napkin should proceed and mark assumptions.

Good behavior:

```text
I’ll assume the top-left screen is the start state and the lower-right screen is the success state.
```

Bad behavior:

```text
Please explain every screen before I continue.
```

### Napkin Should Reason Like a UX Designer

It should consider:

- user goal
- entry point
- exit point
- primary action per screen
- decision points
- error states
- empty states
- screen hierarchy
- repeated patterns
- navigation model
- progressive disclosure
- information architecture
- component consistency

### Napkin Should Reason Like a Design-System Designer

It should normalize rough marks into semantic structures:

| Rough sketch mark | Semantic interpretation |
|---|---|
| Top rectangle | Header / nav bar |
| Three repeated cards | Card list |
| Box with label + underline | Text input |
| Floating rectangle | Modal or popover |
| Horizontal options | Tabs or segmented control |
| Arrow from button to screen | Click transition |

---

## 15. Design-System Mindset

Napkin should maintain a component taxonomy independent of any single visual library.

### Layout

- page
- section
- container
- grid
- stack
- split panel
- sidebar
- header
- footer

### Navigation

- nav bar
- sidebar nav
- tabs
- breadcrumb
- stepper
- pagination
- menu

### Input

- text field
- textarea
- checkbox
- radio group
- select
- combo box
- search field
- switch
- slider

### Actions

- button
- icon button
- link
- menu item
- destructive action
- secondary action

### Content

- card
- list
- table
- avatar
- image placeholder
- heading
- paragraph
- badge
- metadata row

### Feedback

- alert
- toast
- modal
- dialog
- empty state
- loading state
- error state
- confirmation state

### Recommended References

For web UI semantics, Napkin should use headless/component-library thinking as reference material.

Use Radix UI and React Aria as semantic references, not visual styling sources.

Napkin should not “make it look like Radix.” Radix is unstyled. Napkin should instead use Radix-like thinking to decide whether something is a dialog, popover, tab list, menu, form field, or navigation primitive.

### Style-less by Default

Napkin is intentionally **style-less**. Output is grayscale, unbranded, low-fidelity, and uses generic shapes and typography. The reason: Napkin's purpose is to make conversations focus on *user flow and structure*, not visual details. Branded or styled output pulls stakeholders into the wrong discussion at the wrong stage.

Napkin should not:

- apply colors beyond a small grayscale ramp
- apply brand fonts, logos, or imagery
- imitate any specific product's visual language
- generate decorative elements

### User-Provided Design System (Future, Low Priority)

Napkin's MVP does **not** ingest the user's existing design system. This is a deliberate Phase-N+ feature. When it is added later:

- Inputs must conform to **headless-style component specs** (semantic structure, roles, states, slots) — not visual tokens, not CSS, not pixel specs.
- If a user-provided design system spec is incomplete, Napkin must fall back to its built-in headless taxonomy (§15) rather than guessing.
- Even with a custom design system, Napkin's rendering remains low-fi and style-less. The custom design system only informs *which semantic components exist and what they're called*, not *how they look*.

Until that phase ships, design-system inputs are accepted as written context only and do not change the visual output.

### TV / Lean-Back Readiness (Forward-Looking)

TV is a Phase-5 target, but it introduces structural concepts that the IR must be able to express without a breaking schema migration. Phase-1 IR therefore reserves a small set of **optional** fields for TV concepts (see §16). They are not used in MVP rendering and are unenforced — they exist purely so a TV-aware Phase-5 can attach data to the same IR shape.

Concepts being reserved:

- **Focus graph.** TV navigation is a graph (up/down/left/right neighbors), not a hit-test. Phase-1 stores nothing; Phase-5 fills `focusOrder` and/or `focusNeighbors`.
- **Safe area / overscan.** TV frames need ~5% safe-area margins. Reserved per screen.
- **Voice intents.** Voice-controlled actions bind utterances to elements. Reserved per element.
- **Per-screen input model.** Input modality can change *between screens* (a puzzle game's main menu is d-pad-primary; its gameplay screen is voice-primary; its pause overlay is d-pad again). The IR therefore carries an `inputModes` field on both root (default) and screen (override), with explicit `primary` and `also` fields rather than a flat enum.

Recommended **headless semantic references** for TV (analogous to Radix/React Aria for web):

- **Norigin Spatial Navigation** (`@noriginmedia/norigin-spatial-navigation`) — focus-only, no styling.
- **LRUD** (BBC, open source) — left/right/up/down focus primitives.
- **react-tv-space-navigation** — newer alternative.
- **React Native for TV** — built-in focus handling on tvOS / Android TV.

Napkin should treat these as references for *what concepts exist* (focusable, focus group, focus container, rail), not as styling sources.

---

## 16. Napkin Flow IR

Napkin should produce a structured intermediate spec before calling Figma MCP.

### Example Schema

The schema below is the Phase-1 shape. Fields marked **(forward-looking, optional)** are reserved for later phases (mobile, desktop, TV, voice). They MUST be optional and unenforced in Phase-1; their only purpose is to keep the IR shape stable across phases so we don't ship a breaking migration when TV/voice land.

```ts
type NapkinFlowIR = {
  version: "0.2";
  projectName: string;
  targetSurface: "responsive_web" | "mobile_app" | "desktop_app" | "tv";
  fidelity: "low";
  sourceSummary: string;
  assumptions: Assumption[];
  openQuestions: OpenQuestion[];
  screens: NapkinScreen[];
  flows: NapkinFlowEdge[];
  designSystem: DesignSystemMapping;

  // (forward-looking, optional) — flow-wide default input model.
  // Each screen may override.
  inputModes?: InputModeSpec;
};

type NapkinScreen = {
  id: string;
  name: string;
  purpose: string;
  viewport: "desktop" | "tablet" | "mobile" | "tv";
  layoutType: string;
  primaryAction?: string;
  elements: NapkinElement[];
  confidence: number;

  // (forward-looking, optional) — per-screen input override.
  // Example: a puzzle game's gameplay screen sets primary: "voice"
  // even though the rest of the flow is d-pad.
  inputModes?: InputModeSpec;

  // (forward-looking, optional) — TV / lean-back fields.
  focusOrder?: string[];                  // explicit element-id focus order
  safeArea?: {                            // overscan-safe margins, e.g. 5%
    top?: number;
    right?: number;
    bottom?: number;
    left?: number;
  };
};

type NapkinElement = {
  id: string;
  type: ComponentType;
  label?: string;
  role?: string;
  children?: NapkinElement[];
  notes?: string;
  confidence: number;

  // (forward-looking, optional) — voice control.
  // Utterance that targets this element when voice is an active input mode.
  voiceIntent?: string;

  // (forward-looking, optional) — explicit focus neighbors, TV only.
  // If absent, neighbors are derived from layout in Phase-5.
  focusNeighbors?: {
    up?: string;
    down?: string;
    left?: string;
    right?: string;
  };
};

type NapkinFlowEdge = {
  id: string;
  fromScreenId: string;
  fromElementId?: string;
  toScreenId: string;
  trigger: "click" | "submit" | "back" | "select" | "voice" | "unknown";
  label?: string;
  isOptional?: boolean;
};

// (forward-looking) — input modality declaration.
// Used at IR root (default) and per-screen (override).
type InputMode = "pointer" | "touch" | "dpad" | "voice" | "gesture";

type InputModeSpec = {
  primary: InputMode;
  also?: InputMode[];   // additional input modes available on this surface
};
```

### Worked Example: Puzzle Game with Mixed Input

```jsonc
{
  "inputModes": { "primary": "dpad" },          // flow-wide default
  "screens": [
    { "id": "main_menu" /* inherits dpad */ },
    {
      "id": "puzzle_gameplay",
      "inputModes": { "primary": "voice", "also": ["dpad"] },
      "elements": [
        { "id": "answer_field", "type": "voice_prompt",
          "voiceIntent": "the answer is {phrase}" },
        { "id": "pause_btn", "type": "icon_button" }
      ]
    },
    {
      "id": "pause_overlay",
      "inputModes": { "primary": "dpad" }
    }
  ]
}
```

In Phase-1, Napkin parses and persists these fields if a user provides them, but does not validate or render them. In Phase-5, Napkin uses them to drive focus-graph rendering, voice-prompt UI, and clarification questions.

### Why This Matters

Napkin Flow IR lets the system:

- validate before drawing
- ask targeted clarification questions
- regenerate specific screens
- support multiple platforms later
- create a Figma board consistently
- add MCP, plugin, CLI, or web app interfaces later

### IR Persistence and Iteration

Napkin must **persist the IR to disk** in the user's project so that revisions, re-renders, and "catch up" runs are possible across sessions.

**Default persistence path:**

```text
<project-root>/.napkin/
  flow.json          # canonical IR (machine-readable)
  flow.md            # human-readable summary of the IR
  sketches/          # original sketch images, if user dropped them here
  history/           # optional snapshots of prior IR versions
```

**Iteration model:**

When the user asks for revisions ("make screen 3 a modal", "add an error state after signup"), Napkin should:

1. Load the persisted IR from `.napkin/flow.json`.
2. Compute a targeted patch (modify only the affected screens/edges).
3. Save the new IR (and optionally snapshot the old one to `history/`).
4. If Figma MCP is available, re-render only the affected frames on the Figma board — not the whole board.
5. If Figma MCP is unavailable, update the IR files only and let the user re-render later (see §23, "Catch-up rendering").

This diff-based model avoids destroying user edits made directly in Figma where possible, and avoids regenerating the entire board on every small change.

---

## 17. Figma Output Requirements

Napkin should use Figma MCP to create a board with predictable structure.

### Figma Board Structure

```text
Napkin / Project Name
  00 Overview
    Flow Summary
    Assumptions
    Open Questions
    Component Legend

  01 Main Flow
    Screen 1 / Landing
    Screen 2 / Signup
    Screen 3 / Setup Workspace
    Screen 4 / Invite Team
    Screen 5 / Dashboard

  02 Optional Branches
    Optional Invite Flow
    Error / Empty States

  03 Notes
    UX Notes
    Ambiguities
    Suggested Next Steps
```

### Each Screen Frame Should Include

- screen title
- viewport label
- low-fi layout
- semantic layer names
- primary action
- important notes
- consistent spacing
- consistent component treatment

### Flow Graph Should Include

- arrows between screens
- labels for transitions
- optional paths
- start state
- end state
- decision points

### Visual Style

Low-fidelity only:

- grayscale
- simple rectangles
- text labels
- no brand styling
- no visual polish
- minimal icons
- clear hierarchy
- readable stakeholder-facing layout

### Phase-2a Implementation (Official Figma MCP)

Napkin's Phase-2a rendering targets the **official Figma MCP** (`mcp__claude_ai_Figma__*`). All writes route through `use_figma`, which executes arbitrary JS against the Figma Plugin API in a target file.

**Tool mapping:**

| Purpose | Tool |
|---|---|
| Get authenticated user / `planKey` | `mcp__claude_ai_Figma__whoami` |
| Create a new design file | `mcp__claude_ai_Figma__create_new_file` |
| Run Plugin-API JS in a file | `mcp__claude_ai_Figma__use_figma` |
| Verify structure | `mcp__claude_ai_Figma__get_metadata` |
| Verify visually | `mcp__claude_ai_Figma__get_screenshot` |

**File-creation path (Phase 2a):**

- **Default**: user provides an existing Figma file URL on first render. Napkin extracts the `fileKey` and persists it to `figmaBoard.fileKey`.
- **Opt-in**: user explicitly requests a new file. Napkin calls `whoami` then `create_new_file` with `editorType: "design"`.

**Rendering primitives only** in Phase 2a:

- Frames (containers), text, rectangles, lines, polygons (arrowheads).
- No component-library inserts, no auto-layout, no native connectors (FigJam-only).

**Naming conventions** are required on every Napkin-created node:

```
[napkin:screen:<id>] <Name>           # screen frame
[napkin:el:<id>] <type>: <label>      # element node
[napkin:edge:<id>] <from> → <to>      # arrow shaft
[napkin:edge:<id>:head]               # arrow head
[napkin:overview:<kind>]              # overview text frames
```

This naming is the bridge between the IR and Figma — Phase-2b targeted iteration relies on finding nodes by tag.

**Render modes (Phase 2a):**

| Mode | Trigger | Behavior |
|---|---|---|
| First render | `figmaBoard.fileKey` is absent | Resolve fileKey, draw everything, persist metadata |
| Whole-board re-render | User explicitly asks | **Destructive**: delete Napkin-owned pages, redraw. Loses user edits inside Napkin-owned frames. Warn + confirm |
| Targeted iteration | "change screen 3 to a modal" | **Not in 2a.** Update IR; tell user the Figma board needs manual edit or whole-board re-render until 2b |
| Catch-up | "render the latest IR" | Skip interpretation, run rendering pipeline |

**Char-budget caveat:** `use_figma` accepts up to 50,000 chars of JS per call. Napkin batches by **page** (one `use_figma` call per page), splitting further by screen group only if a single page would exceed ~30,000 chars.

Detailed rendering rules and JS recipes are in the Skill at `napkin/references/figma-board-guidelines.md` and `napkin/references/figma-rendering-recipes.md`.

---

## 18. Claude Code Skill Structure

Recommended folder. Phase-1 ships only the files listed under "Phase 1". Per-platform pattern files are stubbed but **deferred** until their corresponding phase — placeholders are left in the structure so the addition is mechanical, not architectural.

```text
napkin/
  SKILL.md
  references/
    # Phase 1 (shipped)
    clarification-questions.md
    ux-interpretation-principles.md
    component-taxonomy.md
    napkin-flow-ir-schema.md

    # Phase 2a (shipped — Figma MCP rendering)
    figma-board-guidelines.md       # board structure, naming, visual rules
    figma-rendering-recipes.md      # Plugin-API JS recipes for primitives + arrows

    # Phase 2b+ (planned)
    examples.md                     # worked IR → Figma examples

    # Phase 3+ (deferred — keep slot, do not write content yet)
    responsive-web-patterns.md   # Phase 3
    mobile-patterns.md           # Phase 4
    desktop-patterns.md          # Phase 5
    tv-patterns.md               # Phase 5
  tests/
    fixtures/
      <sketch>.png
      <sketch>.expected.json     # expected Napkin Flow IR for the sketch
      README.md
```

### A Note on Scripts

Earlier drafts proposed a `scripts/` directory with TypeScript validators (`validate_ir.ts`, `normalize_ir.ts`, `generate_figma_plan.ts`). These have been **dropped from the MVP**.

Reason: Skills execute scripts via shell, and shipping `.ts` files forces the user to install a TypeScript runtime (`tsx`, `bun`, `ts-node`). That is friction without much payoff at MVP.

Replacement plan:

- The IR schema is documented in `references/napkin-flow-ir-schema.md` as JSON Schema + prose.
- The Skill validates the IR by reading the schema reference and reasoning about the proposed IR before saving it.
- If executable validation becomes valuable later, prefer plain Node (no compile step) or Python — pick a runtime the user is likely to already have.

### `SKILL.md` Should Define

- when Napkin should be invoked
- how to inspect sketches
- when to ask clarifying questions
- how to produce Napkin Flow IR
- how to normalize UI patterns
- how to use Figma MCP
- how to structure the Figma board
- how to handle uncertainty
- what not to do

### Recommended Skill Description

```yaml
name: napkin
description: Use this skill when the user wants to convert rough UX sketches, paper sketches, whiteboard photos, screenshots, or early user-flow drawings into structured low-fidelity Figma wireframe boards. Napkin interprets sketches through a UX and design-system mindset, asks clarifying questions when needed, creates a Napkin Flow IR, maps rough UI marks to semantic components, and uses Figma MCP tools to draw editable screens, flow arrows, annotations, assumptions, and shareable user-flow documentation in Figma.
```

---

## 19. Product Roadmap

### Phase 1: Skill-Only POC

Goal:

```text
Sketch image + text context → Napkin Flow IR (persisted to .napkin/)
```

No Figma drawing yet.

Deliverables:

- `SKILL.md` with invocation, image-input handling (inline + path), clarification policy, IR-output policy
- `references/component-taxonomy.md`
- `references/clarification-questions.md`
- `references/ux-interpretation-principles.md`
- `references/napkin-flow-ir-schema.md`
- `tests/fixtures/` with **5–10** sketch + expected-IR pairs and a `README.md` describing how to use them as a regression set

Success criteria:

- Napkin correctly identifies screens and flows on fixture sketches.
- Napkin asks useful clarification questions on ambiguous fixtures and stays quiet on clear ones.
- Napkin produces consistent UX-structured output that round-trips through the documented IR schema.
- Napkin persists the IR to `.napkin/flow.json` and `.napkin/flow.md`.

### Phase 2: Figma MCP Rendering

Goal:

```text
Napkin Flow IR → Figma board
```

Deliverables:

- Figma MCP tool instructions
- frame creation plan
- board layout conventions
- wireframe rendering rules
- flow arrow rendering

Success criteria:

- Figma board is shareable and legible.
- Screens are editable.
- Arrows are readable.
- Naming is consistent.

### Phase 3: Responsive Web Design-System Intelligence

Goal:

```text
Rough sketch → coherent responsive web wireframe system
```

Deliverables:

- responsive web pattern reference
- layout normalization rules
- desktop/mobile variants
- component mapping rules

Success criteria:

- repeated elements become consistent
- layouts feel intentional
- stakeholders can understand the flow without seeing the original sketch

### Phase 4: Mobile App Support

Add:

- native mobile patterns
- tab navigation
- stack navigation
- sheets
- safe areas
- mobile form patterns

### Phase 5: Desktop and TV Support

Desktop:

- sidebars
- dense tables
- multi-pane layouts
- menus
- toolbars

TV:

- d-pad navigation
- focus graph (`focusOrder` / `focusNeighbors` from §16 — already reserved in Phase-1 schema)
- select/back behavior
- carousel/grid patterns (rail, hero billboard, persistent side rail)
- overscan-safe spacing (`safeArea` from §16)
- large typography (24px+ body)
- voice control as a **per-screen primary input** where applicable (`inputModes` + `voiceIntent` from §16 — also reserved in Phase-1 schema)
- voice prompt UI, voice intent legend, character-grid keyboards

TV should not be treated as just "large responsive web." It needs a navigation graph and may have screens where voice — not d-pad — is the primary controller.

Headless semantic references for Phase-5 work: Norigin Spatial Navigation, LRUD, react-tv-space-navigation, React Native for TV. See §15 "TV / Lean-Back Readiness" for context.

---

## 20. Key Product Decisions

### Decision 1: Skill First, Plugin Later Only If Needed

Napkin should start as a Claude Code Skill, not a Figma Plugin.

A Figma Plugin could make sense later if the product needs a one-click Figma-native experience. But Napkin’s first differentiator is AI interpretation, not Figma UI.

### Decision 2: Figma MCP as Execution Layer

Figma MCP should be the drawing and document-generation layer. Napkin should do the thinking, planning, and normalization before calling Figma tools.

### Decision 3: Use a Structured IR

Napkin should always create a structured intermediate representation before drawing. This prevents low-quality direct rendering and makes the product systematic.

### Decision 4: Low-Fidelity Only

Napkin should intentionally generate low-fidelity wireframes and flow diagrams. High-fidelity generation would distract from the actual value.

### Decision 5: Product Name “Napkin”

Napkin works as both a code name and product name. It is memorable, product-relevant, and emotionally connected to the problem: turning rough napkin sketches into structured UX flows.

---

## 21. Naming System

Recommended naming system:

```text
Product: Napkin
Core schema: Napkin Flow IR
Generated board: Napkin Board
Skill package: napkin
Default design preset: Napkin Wireframe System
```

### Tagline Options

- From rough sketch to readable flow.
- Turn napkin sketches into Figma flows.
- Sketch the idea. Napkin structures it.
- Messy sketch in, clean UX flow out.
- Your AI design transcriber for early product flows.
- From paper intent to Figma structure.

Recommended tagline:

```text
Napkin — from rough sketch to readable flow.
```

---

## 22. Success Metrics

### Activation

- Percentage of users who provide sketches and complete a generated Napkin Flow IR.
- Percentage of users who proceed from IR to Figma board generation.
- Time from initial prompt to first usable board.

### Quality

- Percentage of generated screens users keep.
- Number of manual corrections per generated screen.
- User rating of interpretation accuracy.
- User rating of Figma board legibility.
- Component classification accuracy.
- Flow-arrow accuracy.

### Retention

- Repeat usage per user per week.
- Number of projects using Napkin-generated boards.
- Number of regenerated or updated boards.

### North-Star Metric

**Accepted generated screens per user per week.**

“Accepted” means the user keeps, edits, or builds from the generated screen rather than deleting it.

---

## 23. Risks and Mitigations

### Risk: Sketch Recognition Is Unreliable

Mitigations:

- Ask clarifying questions.
- Generate assumptions and open questions.
- Use a structured IR before rendering.
- Start with responsive web patterns.
- Allow user corrections before Figma generation.

### Risk: Output Feels Too Generic

Mitigations:

- Allow product context prompts.
- Support pattern presets such as SaaS onboarding, dashboard, ecommerce, and settings.
- Add design-system mapping over time.
- Preserve important user-provided labels.

### Risk: Figma Output Is Messy

Mitigations:

- Enforce naming conventions.
- Use predictable board sections.
- Use consistent spacing and low-fi visual rules.
- Store metadata where possible.
- Keep rendering intentionally simple.

### Risk: Users Expect High-Fidelity UI

Mitigations:

- Position Napkin as a wireframe and flow-documentation tool.
- Use low-fidelity visual language intentionally.
- Avoid brand styling by default.
- Make “shareable design document” the value proposition.

### Risk: MCP Setup Creates Friction

Mitigations:

- Provide clear setup instructions.
- Keep the Skill usable in **IR-only mode** if Figma MCP is unavailable.
- Separate interpretation from rendering.
- Add diagnostics for MCP connection issues.

#### IR-Only Mode (MCP Unavailable)

When Figma MCP is not connected, Napkin must still complete the interpretation pipeline and deliver a usable artifact:

1. Detect that Figma MCP is unavailable (missing tool, failed handshake, auth error).
2. Tell the user clearly that rendering is skipped and why.
3. **Still write `.napkin/flow.json` and `.napkin/flow.md`** so the user has a shareable, legible record of the flow.
4. Print or surface concrete next steps for setting up Figma MCP.

#### Catch-Up Rendering

After the user fixes their Figma MCP setup, they can run Napkin again with a phrase like:

```text
Catch up — render the latest IR to Figma now.
```

On a catch-up run, Napkin must:

- Load `.napkin/flow.json`.
- Skip re-interpretation (the IR is already approved).
- Render the full board to Figma in one pass.
- Store any board metadata returned by MCP back into `.napkin/flow.json` so future iterative diffs work.

This means a user can sketch, generate the IR, fix MCP setup later, and resume without redoing any interpretation work.

---

## 24. MVP Acceptance Criteria

Napkin MVP is successful when a user can:

1. Provide rough sketches and a short product-context prompt.
2. Receive useful clarification questions only when necessary.
3. Get a structured Napkin Flow IR.
4. Approve or adjust assumptions.
5. Use Figma MCP to generate a shareable Figma board.
6. Edit the generated screens and flow graph in Figma.
7. Share the board with stakeholders as a legible low-fidelity UX artifact.

---

## 25. Recommended Next Deliverable

The next deliverable is the **Phase-1 Skill package**:

```text
napkin/
  SKILL.md
  references/
    component-taxonomy.md
    ux-interpretation-principles.md
    clarification-questions.md
    napkin-flow-ir-schema.md
  tests/
    fixtures/
      README.md
```

`figma-board-guidelines.md`, `examples.md`, and the per-platform pattern files are deferred to their respective phases per §18.

The first implementation milestone should focus on:

```text
Sketch image (inline or file path) + context
  → clarification questions (only when needed)
  → Napkin Flow IR
  → persisted to .napkin/flow.json and .napkin/flow.md
```

Only after the IR quality is strong on the fixture set should Napkin call Figma MCP to draw the board.
