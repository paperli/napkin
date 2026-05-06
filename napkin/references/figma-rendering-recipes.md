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

// Standard sizes keyed by detected frameRatio. Picks orientation + aspect
// to match what the user drew. Preferred over SIZES when frameRatio is set.
const SIZES_BY_RATIO = {
  "9:19.5": [390, 844],   // modern phone, portrait
  "9:16":   [375, 667],   // older phone, portrait
  "3:4":    [768, 1024],  // tablet, portrait
  "4:3":    [1024, 768],  // tablet, landscape
  "16:10":  [1440, 900],  // desktop
  "16:9":   [1440, 810],  // desktop / lean-back
  "21:9":   [2560, 1080], // desktop, ultrawide
  "18:9":   [812, 375],   // rotated phone, landscape
  "1:1":    [600, 600],   // square; ambiguous, should be confirmed in 1b
};

// Fallback when frameRatio is absent (legacy IR or ambiguous detection).
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
  // Prefer detected frameRatio (matches the orientation the user drew);
  // fall back to viewport defaults; finally fall back to desktop.
  const [w, h] =
    (screen.frameRatio && SIZES_BY_RATIO[screen.frameRatio]) ||
    SIZES[screen.viewport] ||
    SIZES.desktop;
  const frame = makeFrame(`[napkin:screen:${screen.id}] ${screen.name}`, w, h);
  frame.strokes = [STROKE_DARK];   // primary content frames use dark stroke
  return frame;
}
```

---

## Recipe 3: Layout screens on canvas

Preserve the relative position of screens as drawn in the sketch. `sketchPosition.x`/`y` are 0..1 normalized centroids inside the source image; the renderer scales them onto a per-source canvas region so a screen drawn top-right in the sketch lands top-right on the Figma board. Sources are stacked vertically with a gap between regions. If `sketchPosition` is missing on any screen (legacy IR), fall back to the previous linear grid.

```js
const GROUP_GAP = 400;

function layoutScreens(parentPage, screens, screenFrames) {
  // Fall back to grid if any screen lacks a position (legacy IR or ambiguous detection).
  if (!screens.every(s => s.sketchPosition)) {
    return layoutScreensGrid(parentPage, screenFrames);
  }

  // Group by source image; screens from the same sketch share one canvas region.
  const groups = new Map();
  screens.forEach((s, i) => {
    const key = s.sketchPosition.sourceSketch || "default";
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key).push({ screen: s, frame: screenFrames[i] });
  });

  let groupY = 0;
  for (const [, members] of groups) {
    // Region big enough that scaled positions read as the user drew them.
    const maxW = Math.max(...members.map(m => m.frame.width));
    const maxH = Math.max(...members.map(m => m.frame.height));
    const canvasW = Math.max(2400, members.length * (maxW + GAP));
    const canvasH = Math.max(1200, Math.ceil(members.length / 3) * (maxH + GAP));

    for (const { screen, frame } of members) {
      const cx = screen.sketchPosition.x * canvasW;
      const cy = screen.sketchPosition.y * canvasH;
      frame.x = cx - frame.width / 2;
      frame.y = groupY + cy - frame.height / 2;
      parentPage.appendChild(frame);
    }

    groupY += canvasH + GROUP_GAP;
  }
}

// Fallback: linear left-to-right with row wrap. Used when sketchPosition is absent.
function layoutScreensGrid(parentPage, screenFrames) {
  screenFrames.forEach((frame, i) => {
    const col = i % COLS_PER_ROW;
    const row = Math.floor(i / COLS_PER_ROW);
    frame.x = col * (frame.width + GAP);
    frame.y = row * (frame.height + GAP);
    parentPage.appendChild(frame);
  });
}
```

Two screens drawn at near-identical positions in the sketch will overlap on the Figma board. Phase-2a accepts this — the user's choice to draw them on top reads as deliberate (e.g. modal-over-parent). Phase-2b can add anti-overlap nudging if the overlap turns out to be unintentional.

---

## Recipe 3.5: Chrome canonicalization

Strongly-semantic chrome elements (`header`, `nav_bar`, `footer`, `sidebar`, `sidebar_nav`) ignore `sketchSize` and snap to a canonical anchor on the parent. The interpreter records only that they exist and what they contain — the renderer applies canonical dimensions. See `ux-interpretation-principles.md § Canonical layout vs. literal marks`.

```js
function isChromeType(t) {
  return t === "header" || t === "footer" || t === "nav_bar"
      || t === "sidebar" || t === "sidebar_nav";
}

// "top" | "bottom" | "left" — canonical edge for a chrome element.
function chromeAnchor(el, viewport) {
  switch (el.type) {
    case "header": return "top";
    case "footer": return "bottom";
    case "sidebar": case "sidebar_nav": return "left";
    case "nav_bar": {
      // Bottom tab bar if drawn in the bottom ~15% of a mobile/tablet screen.
      const y = el.sketchPosition && el.sketchPosition.y;
      const mobileish = viewport === "mobile" || viewport === "tablet";
      return (mobileish && typeof y === "number" && y > 0.85) ? "bottom" : "top";
    }
  }
  return null;
}

// Returns { w, h, x, y } in parent coordinates, or null if not chrome.
function chromeLayout(el, parentNode, viewport) {
  const W = parentNode.width, H = parentNode.height;
  const footerH = viewport === "mobile" ? 56 : 64;
  switch (el.type) {
    case "header":
      return { w: W, h: 56, x: 0, y: 0 };
    case "footer":
      return { w: W, h: footerH, x: 0, y: H - footerH };
    case "sidebar": case "sidebar_nav":
      return { w: 240, h: H, x: 0, y: 0 };
    case "nav_bar":
      return chromeAnchor(el, viewport) === "bottom"
        ? { w: W, h: 64, x: 0, y: H - 64 }
        : { w: W, h: 56, x: 0, y: 0 };
  }
  return null;
}
```

Chrome children (icon slots in a tab bar, links in a header) follow the normal `renderElement` path *inside* the chrome parent — their `sketchPosition` is normalized to the chrome container, not the screen, so they benefit from the chrome's canonical dimensions automatically.

---

## Recipe 4: Component archetypes

The tier-1 archetype helpers (`makeButton`, `makeInput`, `makeToggle`, `makeCard`, `makeListItem`, `makeModal`, `makeAlert`, `makeBadge`, `makeAvatar`, `makeImagePlaceholder`, `makeTextNode`, `makeHeader`, `makeNavBar`, `makeFooter`, `makeIconButton`, `makeCustomShape`) live in `wireframe-library.md`. Paste that block of code into your `use_figma` payload alongside the orchestration helpers and tokens.

The only archetype defined in *this* file is `makeContainer` — the long-tail fallback for tier-2 taxonomy types (`page`, `section`, `container`, `split_panel`, `sidebar`, `tabs`, `stepper`, etc.) that the library hasn't migrated yet.

```js
function makeContainer(el, w, h) {
  const f = makeFrame(`[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`, w, h);
  f.strokes = [STROKE_LIGHT];
  return f;
}
```

---

## Recipe 4.5: Layout elements within a frame

Archetype helpers only create nodes; they don't position them. `renderElement` dispatches to the right archetype, applies `sketchPosition` / `sketchSize` if present, and recurses into `children`.

`sketchPosition.x`/`y` are 0..1 normalized centroids within the parent (the screen frame for top-level elements, or the parent container for nested ones — a button inside a card is normalized to the card). `sketchSize.w`/`h` are 0..1 fractions of the parent's dimensions and override the archetype default size when present.

The dispatch — `createByArchetype` — lives in `wireframe-library.md § Dispatch`. Paste it into the `use_figma` payload alongside the library archetypes. A few archetypes (`makeListItem`, `makeModal`) return `{ node, sibling }` so `renderElement` can append the sibling (divider line, modal scrim) onto the same parent.

```js
// Per-archetype tap-target floors. Clamp absurdly small sketchSize overrides
// so the literal mark size never produces sub-tap-target nodes on mobile.
function archetypeFloor(elType, viewport) {
  const isMobile = viewport === "mobile";
  switch (elType) {
    case "button": case "destructive_action": case "secondary_action":
      return isMobile ? [120, 44] : [80, 32];
    case "icon_button": case "menu_item":
      return isMobile ? [44, 44] : [32, 32];
    case "text_field": case "search_field": case "select":
    case "combo_box": case "textarea":
      return isMobile ? [160, 44] : [120, 32];
    default:
      return [40, 20]; // soft floor for everything else
  }
}

function renderElement(el, parentNode, viewport, fallbackIndex = 0) {
  const created = createByArchetype(el, parentNode, viewport);
  const node    = created && created.node    ? created.node    : created;
  const sibling = created && created.sibling ? created.sibling : null;
  if (sibling) parentNode.appendChild(sibling);

  // Chrome path: canonical size + edge anchor. Ignore sketchSize and sketchPosition.
  if (isChromeType(el.type)) {
    const layout = chromeLayout(el, parentNode, viewport);
    if (layout && typeof node.resize === "function") {
      node.resize(layout.w, layout.h);
      node.x = layout.x;
      node.y = layout.y;
      parentNode.appendChild(node);
      if (el.children && el.children.length) {
        el.children.forEach((child, i) => renderElement(child, node, viewport, i));
      }
      return node;
    }
  }

  // Size: prefer sketchSize, with archetype-aware floors; else keep archetype default.
  // custom_shape honors sketchSize literally (no floor) — that's the whole point of the fallback.
  if (el.sketchSize && typeof node.resize === "function") {
    const [floorW, floorH] = el.type === "custom_shape" ? [8, 8]
                                                        : archetypeFloor(el.type, viewport);
    node.resize(
      Math.max(floorW, el.sketchSize.w * parentNode.width),
      Math.max(floorH, el.sketchSize.h * parentNode.height)
    );
  }

  // Position: prefer sketchPosition centroid; else stack vertically at left margin.
  if (el.sketchPosition) {
    const cx = el.sketchPosition.x * parentNode.width;
    const cy = el.sketchPosition.y * parentNode.height;
    node.x = Math.max(0, cx - node.width / 2);
    node.y = Math.max(0, cy - node.height / 2);
  } else {
    node.x = 24;
    node.y = 24 + fallbackIndex * 56;
  }

  parentNode.appendChild(node);

  // Recurse into children using this node as the new reference frame.
  if (el.children && el.children.length) {
    el.children.forEach((child, i) => renderElement(child, node, viewport, i));
  }

  return node;
}
```

Two elements drawn at near-identical positions overlap on the Figma board. Phase-2a accepts this — same stance as for screens; Phase-2b can add nudging.

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

A complete `use_figma` payload for a small render. Paste in this order: orchestration constants from this file → wireframe library tokens, `autoFrame`, archetype helpers, and dispatch from `wireframe-library.md`.

```js
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });

// ...orchestration constants and helpers from this file...
// ...wireframe-library.md tokens, autoFrame, archetype helpers, createByArchetype...

// Pages
const overview = ensurePage("00 Overview");
const mainFlow = ensurePage("01 Main Flow");

// Overview
renderOverview(overview, ir);

// Screens
const frameMap = {};
const screenFrames = ir.screens.map(s => {
  const f = makeScreenFrame(s);
  s.elements.forEach((el, i) => renderElement(el, f, s.viewport, i));
  frameMap[s.id] = f.id;
  return f;
});
layoutScreens(mainFlow, ir.screens, screenFrames);

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

A typical 5-screen flow (orchestration helpers + library tokens + tier-1 archetypes + dispatch + 5 screens + 4 arrows + overview) lands around 12–15 KB. Comfortably under the 50 KB `use_figma` limit. Larger flows split per page; the helpers and library code re-paste in each call (font state and globals don't carry between calls anyway).

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
