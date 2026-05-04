# Napkin Flow IR — Schema Reference

The Napkin Flow IR is the single source of truth between sketch interpretation and Figma rendering. It is persisted to `.napkin/flow.json` so that revisions, catch-up rendering, and cross-session iteration all work.

This document defines:

1. The TypeScript shape of the IR (v0.2).
2. Validation rules Napkin applies before saving.
3. The persistence format on disk.
4. The `flow.md` human-readable summary template.
5. A worked example.

---

## 1. Schema (v0.2)

Phase-1 ships v0.2. Fields marked **(forward-looking)** are optional and unused in Phase-1 — they exist so Phase-4 (mobile) and Phase-5 (TV / voice) can extend without a breaking migration.

```ts
type NapkinFlowIR = {
  version: "0.2";
  projectName: string;
  targetSurface: "responsive_web" | "mobile_app" | "desktop_app" | "tv";
  fidelity: "low";
  sourceSummary: string;            // 1–3 sentences describing the input
  assumptions: Assumption[];
  openQuestions: OpenQuestion[];
  screens: NapkinScreen[];
  flows: NapkinFlowEdge[];
  designSystem: DesignSystemMapping;

  // (forward-looking) flow-wide default input model.
  // Each screen may override.
  inputModes?: InputModeSpec;

  // (forward-looking) optional metadata Figma MCP fills on first render
  // so iterative diffs can target specific frames.
  figmaBoard?: FigmaBoardMetadata;
};

type NapkinScreen = {
  id: string;                       // stable, snake_case, e.g. "signup"
  name: string;                     // short title, e.g. "Signup"
  purpose: string;                  // one-line job-to-be-done
  viewport: "desktop" | "tablet" | "mobile" | "tv";
  frameRatio?: string;              // detected ratio class, e.g. "9:18", "16:9", "4:3";
                                    // drives renderer frame size + orientation.
                                    // viewport remains the semantic label.
  sketchPosition?: {                // normalized centroid of the drawn frame
    x: number;                      // 0..1, 0 = left edge of source image
    y: number;                      // 0..1, 0 = top edge of source image
    sourceSketch?: string;          // source image filename when multi-image
  };
  layoutType: string;               // freeform short label e.g. "centered_form"
  primaryAction?: string;           // verb describing the screen's main job
  elements: NapkinElement[];
  confidence: number;               // 0..1

  // (forward-looking) per-screen input override
  inputModes?: InputModeSpec;

  // (forward-looking) TV / lean-back fields
  focusOrder?: string[];            // explicit element-id focus sequence
  safeArea?: {                      // overscan margins in px or %
    top?: number;
    right?: number;
    bottom?: number;
    left?: number;
  };
};

type NapkinElement = {
  id: string;                       // stable, snake_case, scoped within a screen
  type: ComponentType;              // from component-taxonomy.md
  label?: string;                   // user-facing text from the sketch
  role?: string;                    // optional ARIA-style role hint
  children?: NapkinElement[];
  notes?: string;                   // free-text designer note
  confidence: number;               // 0..1

  sketchPosition?: {                // normalized centroid within the parent
    x: number;                      // 0..1, relative to parent (screen frame
    y: number;                      // or container) — not the source image
  };
  sketchSize?: {                    // normalized size relative to parent
    w: number;                      // 0..1 fraction of parent width
    h: number;                      // 0..1 fraction of parent height
  };

  // (forward-looking) voice control
  voiceIntent?: string;             // utterance that targets this element

  // (forward-looking) TV focus neighbors. If absent in Phase-5,
  // neighbors are derived from layout.
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
  fromElementId?: string;           // omit if transition is from screen-as-a-whole
  toScreenId: string;
  trigger: "click" | "submit" | "back" | "select" | "voice" | "unknown";
  label?: string;                   // short label like "Sign up"
  isOptional?: boolean;
};

type Assumption = {
  id: string;
  text: string;                     // human-readable sentence
  appliesTo?: { screenId?: string; elementId?: string; edgeId?: string };
};

type OpenQuestion = {
  id: string;
  text: string;
  blocksRender?: boolean;           // true = Napkin should not render until answered
  appliesTo?: { screenId?: string; elementId?: string; edgeId?: string };
};

type DesignSystemMapping = {
  // Phase-1: Napkin's own taxonomy is the only source.
  // Future phases may accept user-provided headless component specs here.
  source: "napkin_default";
  taxonomyVersion: string;          // e.g. "0.2"
  notes?: string;
};

// (forward-looking) input modality declaration.
// Used at IR root (default) and per-screen (override).
type InputMode = "pointer" | "touch" | "dpad" | "voice" | "gesture";

type InputModeSpec = {
  primary: InputMode;
  also?: InputMode[];
};

type FigmaBoardMetadata = {
  fileKey?: string;
  pageId?: string;
  frameIds?: { [screenId: string]: string };
  lastRenderedAt?: string;          // ISO timestamp
};

// ComponentType is the union of strings listed in component-taxonomy.md.
// Phase-1 enforcement: must match a known type from that file.
type ComponentType = string;
```

---

## 2. Validation rules

Napkin validates the IR by reasoning against this schema **before** writing it to disk. There is no executable validator in Phase-1 (per PRD §18 the script-based validators were dropped). The Skill is responsible for these checks:

### Structural

- `version` is `"0.2"`.
- Every `screens[].id` is unique.
- Every `flows[].fromScreenId` and `flows[].toScreenId` references an existing screen.
- Every `flows[].fromElementId`, when present, references an element on `fromScreenId`.
- Every element `id` is unique within its screen's element subtree.
- Every `assumptions[].appliesTo` and `openQuestions[].appliesTo` reference real ids if provided.

### Semantic

- Every `NapkinElement.type` is a valid `ComponentType` from `component-taxonomy.md`. If it isn't, either remap to the closest valid type or surface as an open question — do not invent.
- Every screen has at least one element.
- The flow graph has at least one screen with no incoming edges (the entry point) **or** the IR contains an `assumption` explicitly naming the entry point.
- Every `confidence` is in `[0, 1]`.
- `targetSurface` matches the dominant viewport across screens (warn if not).

### Forward-looking

- If `inputModes.primary` is `"voice"` on any screen, every actionable element on that screen should have a `voiceIntent` — if not, add an open question.
- If `targetSurface` is `"tv"`, every screen should have a `focusOrder` or be implicitly focus-orderable. Phase-1 does not enforce this; flag in `openQuestions` instead.

### What to do on validation failure

- Hard structural failures (orphan reference, duplicate id) → fix before writing. The Skill must not save an IR it cannot self-validate.
- Soft semantic failures (low confidence, missing entry point) → write an `openQuestion` and proceed.

---

## 3. Persistence format

```
<project-root>/.napkin/
  flow.json     # canonical IR — pretty-printed JSON, 2-space indent
  flow.md       # human-readable summary (template below)
  sketches/     # original sketch images (copies are fine)
  history/      # snapshots of prior IR versions, named <ISO-timestamp>.json
```

Rules:

- `flow.json` is the **source of truth**. `flow.md` is regenerated from it on every save.
- `history/` is append-only. Never delete.
- Snapshots are taken **before** overwriting `flow.json`, not after.
- Use ISO 8601 timestamps for snapshot filenames: `2026-05-01T13-22-45Z.json` (colons replaced with dashes for filesystem safety).

---

## 4. `flow.md` template

`flow.md` is a stakeholder-friendly summary. It does not contain anything not derivable from `flow.json`.

```markdown
# {{projectName}}

**Target surface:** {{targetSurface}}
**Fidelity:** {{fidelity}}
**Generated:** {{ISO timestamp}}
**IR version:** {{version}}

## Source

{{sourceSummary}}

## Screens

### {{screen.name}} ({{screen.id}})

**Purpose:** {{screen.purpose}}
**Viewport:** {{screen.viewport}}
**Primary action:** {{screen.primaryAction}}
**Confidence:** {{screen.confidence}}

Elements:

- {{element.type}} — {{element.label or '(no label)'}} ({{element.confidence}})
  ...nested as a tree

(repeat per screen)

## Flow

- {{edge.fromScreenId}}.{{edge.fromElementId}} — {{edge.trigger}} → {{edge.toScreenId}}{{ ' (optional)' if edge.isOptional }}

(repeat per edge)

## Assumptions

- {{assumption.text}} (applies to {{appliesTo}})

## Open questions

- {{question.text}} (applies to {{appliesTo}}){{ ' [BLOCKING]' if blocksRender }}
```

---

## 5. Worked example: puzzle game with mixed input

Demonstrates the forward-looking fields. Phase-1 won't generate this exact shape from a sketch (TV is Phase-5), but the IR shape supports it today, which is the point.

```jsonc
{
  "version": "0.2",
  "projectName": "Wordle TV",
  "targetSurface": "tv",
  "fidelity": "low",
  "sourceSummary": "User sketched a 3-screen TV puzzle game: main menu, gameplay (voice answer), pause overlay.",
  "inputModes": { "primary": "dpad" },
  "assumptions": [
    { "id": "a1", "text": "Main menu is the entry point." }
  ],
  "openQuestions": [
    { "id": "q1", "text": "Should the pause overlay also support a voice 'pause' command?", "blocksRender": false, "appliesTo": { "screenId": "pause_overlay" } }
  ],
  "screens": [
    {
      "id": "main_menu",
      "name": "Main Menu",
      "purpose": "Choose a puzzle pack or resume play.",
      "viewport": "tv",
      "layoutType": "centered_list",
      "primaryAction": "Start a puzzle",
      "confidence": 0.9,
      "elements": [
        { "id": "title", "type": "heading", "label": "Wordle TV", "confidence": 0.95 },
        { "id": "btn_play", "type": "button", "label": "Play", "confidence": 0.95 },
        { "id": "btn_resume", "type": "button", "label": "Resume", "confidence": 0.85 },
        { "id": "btn_settings", "type": "button", "label": "Settings", "confidence": 0.9 }
      ]
    },
    {
      "id": "puzzle_gameplay",
      "name": "Puzzle Gameplay",
      "purpose": "Answer the puzzle by speaking.",
      "viewport": "tv",
      "layoutType": "stage",
      "primaryAction": "Speak the answer",
      "confidence": 0.85,
      "inputModes": { "primary": "voice", "also": ["dpad"] },
      "elements": [
        { "id": "puzzle_prompt", "type": "heading", "label": "Five letters. Starts with C.", "confidence": 0.9 },
        {
          "id": "answer_field",
          "type": "search_field",
          "label": "Speak your answer",
          "voiceIntent": "the answer is {phrase}",
          "confidence": 0.8
        },
        { "id": "pause_btn", "type": "icon_button", "label": "Pause", "confidence": 0.9 }
      ]
    },
    {
      "id": "pause_overlay",
      "name": "Pause",
      "purpose": "Pause, resume, or quit.",
      "viewport": "tv",
      "layoutType": "modal",
      "primaryAction": "Resume",
      "confidence": 0.9,
      "inputModes": { "primary": "dpad" },
      "elements": [
        { "id": "btn_resume", "type": "button", "label": "Resume", "confidence": 0.95 },
        { "id": "btn_quit", "type": "destructive_action", "label": "Quit", "confidence": 0.9 }
      ]
    }
  ],
  "flows": [
    { "id": "e1", "fromScreenId": "main_menu", "fromElementId": "btn_play", "toScreenId": "puzzle_gameplay", "trigger": "select", "label": "Play" },
    { "id": "e2", "fromScreenId": "puzzle_gameplay", "fromElementId": "pause_btn", "toScreenId": "pause_overlay", "trigger": "select", "label": "Pause" },
    { "id": "e3", "fromScreenId": "pause_overlay", "fromElementId": "btn_resume", "toScreenId": "puzzle_gameplay", "trigger": "select", "label": "Resume" }
  ],
  "designSystem": {
    "source": "napkin_default",
    "taxonomyVersion": "0.2"
  }
}
```

---

## 6. Backwards compatibility

If a future IR ships at v0.3, the loader should:

- Reject anything earlier than `0.2` (Phase-1 is the floor).
- Accept newer minor versions if they only add optional fields.
- Refuse to load a major-version bump and prompt the user to regenerate.
