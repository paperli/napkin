# Napkin

> From rough sketch to readable flow.

Napkin is a [Claude Code](https://claude.com/claude-code) Skill that turns rough product-flow sketches — paper drawings, whiteboard photos, screenshots — into structured low-fidelity Figma wireframe boards.

The thesis: **transcribe design intent, not pixels.** Napkin reads sketches through a UX and design-system lens, asks clarifying questions when something is genuinely ambiguous, and produces a clean, editable artifact you can share with stakeholders.

Output is intentionally **style-less** (grayscale, unbranded, low-fi) so the conversation stays on flow and structure — not on visual details that don't matter at this stage.

---

## How it works

```
Sketches + product context
   ↓
Claude Code Skill asks clarifying UX questions (only when needed)
   ↓
Skill interprets screens, flows, and UI intent
   ↓
Skill writes a structured Napkin Flow IR to ./.napkin/
   ↓
Skill calls Figma MCP (Phase 2+)
   ↓
Figma board with clean low-fi wireframes and a flow graph
```

Napkin's brain is the Skill (interpretation, normalization, design-system reasoning). Figma MCP is the drawing hand. Everything is structured around an intermediate representation called **Napkin Flow IR** so revisions, catch-up rendering, and cross-session iteration all work without re-interpreting the sketch.

---

## Status

**Phase 2a — IR + Figma rendering (current).**

Napkin now interprets sketches, persists a structured IR to `.napkin/`, and renders a low-fi wireframe board to a Figma file via the official Figma MCP. Targeted iteration ("change just screen 3 to a modal") is Phase 2b — for now, revisions are made by re-rendering the whole board or hand-editing in Figma.

Roadmap:

| Phase | Scope |
|---|---|
| 1   | Sketch → Napkin Flow IR (responsive web, persisted to `.napkin/`) ✅ |
| 2a  | IR → Figma board via official Figma MCP (first render + whole-board re-render) ✅ |
| 2b  | Targeted iteration (patch single screen / element / edge in place) |
| 3   | Responsive-web design-system intelligence (consistent layouts, mobile/desktop variants) |
| 4   | Mobile app patterns (sheets, tab bars, safe areas) |
| 5   | Desktop and TV (focus graphs, voice-primary screens, lean-back UX) |

See [`docs/PRD.md`](docs/PRD.md) for the full product spec, schema, and roadmap.

---

## Install

Napkin is a Claude Code Skill. Install it by symlinking the `napkin/` directory into your Claude Code skills directory.

### User-level (recommended — available in every project)

```bash
git clone git@github.com:paperli/napkin.git ~/code/napkin
ln -s ~/code/napkin/napkin ~/.claude/skills/napkin
```

### Project-level

```bash
mkdir -p .claude/skills
ln -s /absolute/path/to/napkin/napkin .claude/skills/napkin
```

Restart Claude Code (or start a new session) so the Skill is discovered.

### Requirements

- [Claude Code](https://claude.com/claude-code) installed.
- For Figma rendering (Phase 2a+): the **official Figma MCP** connected and authenticated. Without it, Napkin runs in IR-only mode and writes `.napkin/flow.json` + `.napkin/flow.md` for you to render later.

---

## Quick start

In a Claude Code session, point Napkin at a sketch:

**Inline image:**

> Use Napkin to turn this sketch into a responsive-web SaaS onboarding flow. *(attach image)*

**File path:**

> I have sketches at `~/Desktop/onboarding/`. Use Napkin to turn them into a responsive-web flow.

Napkin will:

1. Read the sketch(es).
2. Ask clarifying questions only if something is genuinely ambiguous (it tries not to over-ask).
3. Produce a Napkin Flow IR and save it to `./.napkin/flow.json` and `./.napkin/flow.md` in the current project.
4. If the official Figma MCP is connected, render the IR to a Figma board. On the first render Napkin will ask for a Figma file URL — paste an empty design file you've created, and Napkin will draw the board into it. From then on, the file is remembered.

The `.napkin/` folder is the source of truth for revisions. The IR also stores the Figma file id and per-screen frame ids, so future sessions resume against the same Figma file without re-asking.

---

## What you get

```
<your-project>/.napkin/
  flow.json     # canonical IR — machine-readable
  flow.md       # human-readable summary of the flow
  sketches/     # original sketch images
  history/      # snapshots of prior IR versions
```

`flow.md` is shareable as-is for stakeholder review. `flow.json` is the input for Figma rendering and future tooling.

---

## Project layout

```
napkin/
├── README.md                 (this file)
├── docs/
│   └── PRD.md                full product requirements + schema spec
└── napkin/                   the Claude Code Skill itself
    ├── SKILL.md              workflow + decision points
    ├── references/
    │   ├── ux-interpretation-principles.md
    │   ├── component-taxonomy.md
    │   ├── clarification-questions.md
    │   ├── napkin-flow-ir-schema.md
    │   ├── figma-board-guidelines.md       (Phase 2a)
    │   └── figma-rendering-recipes.md      (Phase 2a)
    └── tests/
        └── fixtures/
            └── README.md     fixture format + recommended set
```

`SKILL.md` carries only the workflow. Detailed knowledge (component vocabulary, clarification patterns, the full IR schema, interpretation principles) lives in `references/` and is loaded by the Skill on demand.

---

## Design principles

- **Transcribe intent, not marks.** A rectangle in a sketch is a header, a card, a modal, or an input — depending on its role. Napkin's job is to recover the role, not to copy the rectangle.
- **Style-less by default.** Grayscale, unbranded, low-fi. The point is to keep stakeholder discussion on flow and structure.
- **Clarify only when material.** Questions are reserved for ambiguity that would change the output. Cosmetic uncertainty becomes an assumption.
- **Persist everything.** The IR lives on disk. Revisions diff against it. Figma rendering is a separate, deferrable step.

---

## Contributing

Issues and PRs welcome. The project is early — the most valuable contributions right now are:

- Sketch + expected-IR fixtures (see `napkin/tests/fixtures/README.md`).
- Edge cases you hit when running Napkin against your own sketches.
- Notes on where the component taxonomy or clarification patterns fall short.

---

## License

TBD — to be confirmed before public use.

---

## Credits

Built as a [Claude Code](https://claude.com/claude-code) Skill, designed to work with [Figma MCP](https://github.com/figma/figma-mcp-server) as its rendering layer.
