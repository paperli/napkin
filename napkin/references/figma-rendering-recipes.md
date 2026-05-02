# Figma Rendering Recipes

Concrete Plugin-API JS patterns for rendering the Napkin Flow IR onto a Figma file. Used by `SKILL.md` Step 5 in conjunction with the structural rules in `figma-board-guidelines.md`.

All recipes here run inside the `code` parameter of `mcp__claude_ai_Figma__use_figma`. The Figma Plugin API global is `figma`. The agent **adapts** these patterns to the IR being rendered — copy verbatim only the constants and helpers.

---

## Shape of every `use_figma` payload

```js
// 1. Load fonts (always — font state is per call, not per file)
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });

// 2. Define helpers and constants (paste from this file)
// 3. Do the rendering work
// 4. Return JSON-serializable result for Napkin to capture
return {
  pageId: <main page id>,
  frameIds: { /* screenId → figmaFrameId */ },
  lastRenderedAt: new Date().toISOString(),
};
```

Napkin reads the return value and writes it back to `.napkin/flow.json:figmaBoard`. The `fileKey` is already known to Napkin — the JS does not need to return it.

---

## Style constants (paste once per payload)

```js
const STROKE_LIGHT = { type: "SOLID", color: { r: 0.898, g: 0.898, b: 0.898 } }; // #E5E5E5
const STROKE_DARK  = { type: "SOLID", color: { r: 0.2,   g: 0.2,   b: 0.2   } }; // #333333
const FILL_BUTTON  = { type: "SOLID", color: { r: 0.961, g: 0.961, b: 0.961 } }; // #F5F5F5
const FILL_WHITE   = { type: "SOLID", color: { r: 1,     g: 1,     b: 1     } };
const TEXT_DARK    = { type: "SOLID", color: { r: 0.2,   g: 0.2,   b: 0.2   } };
const TEXT_MUTED   = { type: "SOLID", color: { r: 0.467, g: 0.467, b: 0.467 } }; // #777777

const SIZES = {
  desktop: [1440, 900],
  tablet:  [768, 1024],
  mobile:  [375, 812],
  tv:      [1920, 1080],
};

const GAP = 200;
const COLS_PER_ROW = 4;
```

---

## Helper functions

```js
function makeFrame(name, w, h) {
  const f = figma.createFrame();
  f.name = name;
  f.resize(w, h);
  f.fills = [FILL_WHITE];
  f.strokes = [STROKE_LIGHT];
  f.strokeWeight = 1;
  f.cornerRadius = 0;
  return f;
}

function makeText(content, size = 14, weight = "regular") {
  const t = figma.createText();
  t.fontName = { family: "Inter", style: weight === "bold" ? "Bold" : "Regular" };
  t.characters = content;
  t.fontSize = size;
  t.fills = [TEXT_DARK];
  return t;
}

function ensurePage(name) {
  let page = figma.root.children.find(p => p.name === name);
  if (!page) {
    page = figma.createPage();
    page.name = name;
  }
  return page;
}

// Find a Napkin-owned node by its [napkin:...] tag
function findByNapkinTag(root, tag) {
  return root.findOne(n => typeof n.name === "string" && n.name.includes(tag));
}
```

---

## Recipe 1: Page setup

```js
const overview      = ensurePage("00 Overview");
const mainFlow      = ensurePage("01 Main Flow");
// Optional pages — create only if needed
const optBranches   = ir.flows.some(e => e.isOptional) ? ensurePage("02 Optional Branches") : null;
const hasNotes      = (ir.assumptions?.length ?? 0) > 0 || (ir.openQuestions?.length ?? 0) > 0;
const notes         = hasNotes ? ensurePage("03 Notes") : null;
```

---

## Recipe 2: Screen frame

```js
function makeScreenFrame(screen) {
  const [w, h] = SIZES[screen.viewport] || SIZES.desktop;
  const frame = makeFrame(`[napkin:screen:${screen.id}] ${screen.name}`, w, h);
  frame.strokes = [STROKE_DARK];   // primary content frames use dark stroke
  return frame;
}
```

---

## Recipe 3: Layout screens on canvas

Linear left-to-right, wrap every 4 screens, 200 px gutter.

```js
function layoutScreens(parentPage, screenFrames) {
  screenFrames.forEach((frame, i) => {
    const col = i % COLS_PER_ROW;
    const row = Math.floor(i / COLS_PER_ROW);
    frame.x = col * (frame.width + GAP);
    frame.y = row * (frame.height + GAP);
    parentPage.appendChild(frame);
  });
}
```

---

## Recipe 4: Component archetypes

Each archetype renders a small group of related component types. Position is set by the parent's layout — these helpers just create the node.

### Container archetype

For: `page`, `section`, `container`, `header`, `footer`, `sidebar`, `split_panel`, `card`, `modal`, `dialog`.

```js
function makeContainer(el, w, h) {
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, w, h);
  f.strokes = [STROKE_LIGHT];
  return f;
}
```

A `modal` or `dialog` should be drawn smaller than the parent screen and centered, with an optional 50%-opacity dark backdrop frame behind it.

### Text archetype

For: `heading`, `paragraph`, `link`, `badge`.

```js
function makeTextNode(el) {
  const sizes   = { heading: 24, paragraph: 14, link: 14, badge: 12 };
  const weights = { heading: "bold", badge: "bold" };
  const t = makeText(el.label || "(text)", sizes[el.type] || 14, weights[el.type]);
  t.name = `[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`;
  return t;
}
```

### Action archetype

For: `button`, `icon_button`, `menu_item`, `destructive_action`, `secondary_action`.

```js
function makeButton(el) {
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, 120, 36);
  f.fills = [FILL_BUTTON];
  f.strokes = [STROKE_DARK];
  f.cornerRadius = 4;
  const label = makeText(el.label || el.type, 14);
  label.x = 16;
  label.y = 8;
  f.appendChild(label);
  return f;
}
```

### Input archetype

For: `text_field`, `textarea`, `search_field`, `select`, `combo_box`.

```js
function makeInput(el) {
  const isMultiline = el.type === "textarea";
  const w = 280, h = isMultiline ? 96 : 36;
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, w, h);
  f.strokes = [STROKE_DARK];
  f.cornerRadius = 4;
  const placeholder = makeText(el.label || el.type, 14);
  placeholder.fills = [TEXT_MUTED];
  placeholder.x = 12;
  placeholder.y = 8;
  f.appendChild(placeholder);
  return f;
}
```

### Toggle archetype

For: `checkbox`, `radio_group`, `switch`.

```js
function makeToggle(el) {
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, 200, 24);
  f.fills = [];
  f.strokes = [];
  const ind = figma.createRectangle();
  ind.resize(16, 16);
  ind.x = 0; ind.y = 4;
  ind.strokes = [STROKE_DARK];
  ind.cornerRadius = el.type === "switch" ? 8 : 2;
  f.appendChild(ind);
  const label = makeText(el.label || el.type, 14);
  label.x = 24; label.y = 4;
  f.appendChild(label);
  return f;
}
```

### Media archetype

For: `avatar`, `image_placeholder`.

```js
function makeMedia(el) {
  const isAvatar = el.type === "avatar";
  const w = isAvatar ? 40 : 240, h = isAvatar ? 40 : 160;
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, w, h);
  f.strokes = [STROKE_DARK];
  f.cornerRadius = isAvatar ? w / 2 : 4;
  return f;
}
```

### Feedback archetype

For: `alert`, `toast`, `empty_state`, `loading_state`, `error_state`, `confirmation_state`.

```js
function makeFeedback(el) {
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, 320, 64);
  f.strokes = [STROKE_DARK];
  f.cornerRadius = 4;
  const label = makeText(el.label || `(${el.type})`, 14);
  label.x = 16; label.y = 22;
  f.appendChild(label);
  return f;
}
```

### Navigation archetype

For: `nav_bar`, `sidebar_nav`, `tabs`, `breadcrumb`, `stepper`, `pagination`, `menu`.

Phase-2a simplifies aggressively. A `nav_bar` is a horizontal frame with a brand label on the left.

```js
function makeNavBar(el, parentWidth) {
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, parentWidth, 56);
  f.strokes = [STROKE_LIGHT];
  const brand = makeText(el.label || "Brand", 16, "bold");
  brand.x = 16; brand.y = 18;
  f.appendChild(brand);
  return f;
}
```

For the other navigation types in Phase 2a, use the container archetype and place text labels at sensible offsets — the goal is a recognizable hint of structure, not pixel-perfect navigation.

---

## Recipe 5: Flow arrow

Straight line from source frame center to destination frame center, with a triangle arrowhead.

```js
function makeArrow(edge, fromFrame, toFrame, parentPage) {
  const fx = fromFrame.x + fromFrame.width  / 2;
  const fy = fromFrame.y + fromFrame.height / 2;
  const tx = toFrame.x   + toFrame.width    / 2;
  const ty = toFrame.y   + toFrame.height   / 2;

  const dx = tx - fx, dy = ty - fy;
  const len = Math.sqrt(dx * dx + dy * dy);
  const angleDeg = Math.atan2(dy, dx) * 180 / Math.PI;

  // Shaft
  const line = figma.createLine();
  line.name = `[napkin:edge:${edge.id}] ${edge.fromScreenId} → ${edge.toScreenId}`;
  line.x = fx;
  line.y = fy;
  line.resize(len, 0);
  line.rotation = -angleDeg;
  line.strokes = [STROKE_DARK];
  line.strokeWeight = 1;
  if (edge.isOptional) line.dashPattern = [4, 2];
  parentPage.appendChild(line);

  // Arrowhead
  const head = figma.createPolygon();
  head.pointCount = 3;
  head.resize(8, 12);
  head.x = tx - 4;
  head.y = ty - 6;
  head.rotation = -angleDeg - 90;
  head.fills = [STROKE_DARK];
  head.strokes = [];
  head.name = `[napkin:edge:${edge.id}:head]`;
  parentPage.appendChild(head);

  // Label
  if (edge.label) {
    const lbl = makeText(edge.label + (edge.isOptional ? " (optional)" : ""), 12);
    lbl.x = (fx + tx) / 2;
    lbl.y = (fy + ty) / 2 - 16;
    parentPage.appendChild(lbl);
  }

  return line;
}
```

Phase-2a accepts visually approximate arrows that connect "near enough." The math for endpoint placement on the source/destination boundary (instead of center-to-center) is Phase-2b polish.

---

## Recipe 6: Overview page content

The `00 Overview` page is plain text frames. Lay them out vertically with 200 px gutters at `x = 80`.

```js
function renderOverview(page, ir) {
  let y = 80;

  const summary = makeText(`${ir.projectName}\n\n${ir.sourceSummary}`, 14);
  summary.name = "[napkin:overview:summary]";
  summary.x = 80; summary.y = y;
  page.appendChild(summary);
  y += 240;

  if (ir.assumptions?.length) {
    const a = makeText(
      "Assumptions\n\n" + ir.assumptions.map(x => `• ${x.text}`).join("\n"),
      14
    );
    a.name = "[napkin:overview:assumptions]";
    a.x = 80; a.y = y;
    page.appendChild(a);
    y += 240;
  }

  if (ir.openQuestions?.length) {
    const q = makeText(
      "Open questions\n\n" + ir.openQuestions.map(x =>
        `• ${x.text}${x.blocksRender ? " [BLOCKING]" : ""}`
      ).join("\n"),
      14
    );
    q.name = "[napkin:overview:open_questions]";
    q.x = 80; q.y = y;
    page.appendChild(q);
  }
}
```

Use the same pattern on `03 Notes` if you split the notes across pages.

---

## Recipe 7: Whole-board re-render

When the user explicitly requests a rebuild, delete Napkin-owned pages first, then re-create.

```js
function deleteNapkinOwnedPages() {
  const targets = ["00 Overview", "01 Main Flow", "02 Optional Branches", "03 Notes"];
  figma.root.children
    .filter(p => targets.includes(p.name))
    .forEach(p => p.remove());
}
```

Always warn the user before this runs — user edits inside Napkin-owned frames are lost.

---

## Putting it together

A complete `use_figma` payload for a small render:

```js
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });

// ...constants and helpers (paste from above)...

// Pages
const overview = ensurePage("00 Overview");
const mainFlow = ensurePage("01 Main Flow");

// Overview
renderOverview(overview, ir);

// Screens
const frameMap = {};
const screenFrames = ir.screens.map(s => {
  const f = makeScreenFrame(s);
  // ...recursively render s.elements into f using archetype helpers...
  frameMap[s.id] = f.id;
  return f;
});
layoutScreens(mainFlow, screenFrames);

// Arrows
ir.flows.forEach(edge => {
  const from = screenFrames.find(f => f.name.startsWith(`[napkin:screen:${edge.fromScreenId}]`));
  const to   = screenFrames.find(f => f.name.startsWith(`[napkin:screen:${edge.toScreenId}]`));
  if (from && to) makeArrow(edge, from, to, mainFlow);
});

return {
  pageId: mainFlow.id,
  frameIds: frameMap,
  lastRenderedAt: new Date().toISOString(),
};
```

---

## Char-count budget

A typical 5-screen flow (helpers + 5 screens + 4 arrows + overview) lands around 5–8 KB. Comfortably under the 50 KB `use_figma` limit.

For larger flows (15+ screens), split into per-page calls. The helpers can be re-pasted in each call — the duplication is fine and avoids state coupling between calls.

---

## Failure modes

| Failure | Recovery |
|---|---|
| `loadFontAsync` fails | Inter is the Figma default; if it fails, fall back to system default |
| Generated JS exceeds 50 KB | Split: render per page first, then per screen group |
| `use_figma` returns an error | Read the error message, regenerate JS with the fix, retry. Surface to the user — do not silently swallow |
| Plugin-API call throws (invalid color, missing field) | Wrap risky sections in try/catch in the JS; return `{ error: e.message }` so Napkin can report cleanly |
| Frame missing after render | Use `get_metadata` to confirm; if structurally broken, offer a whole-board re-render |

---

## What this file does NOT cover

- **Component-library inserts.** Phase-2a is primitives only. No `figma.importComponentByKeyAsync` calls.
- **Auto-layout.** Phase-2a uses fixed positions. Auto-layout is Phase-2b.
- **Cross-page arrows.** Phase-2a draws each page's arrows on the page where both endpoints exist.
- **Element-level diffs.** Phase-2a does whole-frame and whole-board operations only. Element-level patching is Phase-2b.
- **Code Connect mappings.** Out of scope for Napkin — that's a separate tool ecosystem.
