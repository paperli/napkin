# Wireframe Library

A headless, low-fi component library that turns Napkin IR elements into consistent Figma frames. The taxonomy in `component-taxonomy.md` says *what* something is; this library says *how it's drawn*.

"Headless" here means: structure, behavior conventions, and minimal low-fi style only. No brand colors, no shadows, no typography choices beyond Inter, no cues drawn from the user's design system. Phase-1 stance.

Tier-1 scope — the components below have first-class render functions. Long-tail taxonomy types fall back to the generic container archetype until they're migrated.

```
Tier 1: button, icon_button, text_field/textarea/search_field/select/combo_box,
        checkbox/radio_group/switch, card, list_item, modal/dialog,
        alert/toast/empty_state/loading_state/error_state/confirmation_state,
        badge, avatar, image_placeholder, heading/paragraph/link,
        header, nav_bar, footer, custom_shape

Tier 2 (later): page, section, container, split_panel, sidebar, sidebar_nav,
        tabs, breadcrumb, stepper, pagination, menu, slider, list, table,
        metadata_row, grid, stack
```

Render functions are JS pasted into a `use_figma` payload along with the orchestration code in `figma-rendering-recipes.md`. Each takes `(el, parentNode, viewport)` and returns a Figma node. The caller (`renderElement` in the recipes file) handles position and `sketchSize` overrides; most components use auto-layout internally so labels stay correctly aligned when the parent resizes them.

---

## Tokens

Single source of truth for spacing, type, radius, and color. Paste once per `use_figma` payload along with the orchestration code from `figma-rendering-recipes.md`.

```js
// Color ramp — low-fi grayscale only
const C_INK         = { type: "SOLID", color: { r: 0.2,   g: 0.2,   b: 0.2   } }; // #333333
const C_INK_MUTED   = { type: "SOLID", color: { r: 0.467, g: 0.467, b: 0.467 } }; // #777777
const C_LINE        = { type: "SOLID", color: { r: 0.898, g: 0.898, b: 0.898 } }; // #E5E5E5
const C_FILL_BTN    = { type: "SOLID", color: { r: 0.961, g: 0.961, b: 0.961 } }; // #F5F5F5
const C_FILL_DIS    = { type: "SOLID", color: { r: 0.98,  g: 0.98,  b: 0.98  } }; // #FAFAFA
const C_WHITE       = { type: "SOLID", color: { r: 1,     g: 1,     b: 1     } };
const C_SCRIM       = { type: "SOLID", color: { r: 0,     g: 0,     b: 0     }, opacity: 0.4 };

// Type scale (px). Single typeface = Inter.
const T = { heading: 24, h2: 18, body: 14, caption: 12 };

// Spacing scale (px). Use S[1]..S[8]; never raw numbers.
const S = { 1: 4, 2: 8, 3: 12, 4: 16, 5: 24, 6: 32, 7: 48, 8: 64 };

// Radius scale (px).
const R = { none: 0, sm: 4, md: 8, lg: 12, xl: 16, full: 9999 };

// Tap-target floors.
const TAP = { mobile: 44, desktop: 32 };
```

These supersede the older `STROKE_*`, `FILL_*`, `TEXT_*` names from `figma-rendering-recipes.md`. Use these for any new code; the legacy aliases remain only for orchestration code that hasn't migrated.

---

## Auto-layout helper

Most components are auto-layout frames. This one helper sets the conventional flags so call sites stay focused on what's specific to each component.

```js
function autoFrame({ name, dir = "HORIZONTAL", pad = S[4], gap = S[2], align = "CENTER" }) {
  const f = figma.createFrame();
  f.name = name;
  f.layoutMode = dir;
  f.primaryAxisAlignItems = align;       // CENTER | MIN | MAX | SPACE_BETWEEN
  f.counterAxisAlignItems = "CENTER";
  f.paddingLeft = pad; f.paddingRight = pad; f.paddingTop = pad; f.paddingBottom = pad;
  f.itemSpacing = gap;
  f.primaryAxisSizingMode = "FIXED";     // FIXED so external resize() works
  f.counterAxisSizingMode = "FIXED";
  f.fills = [];
  return f;
}
```

After `autoFrame`, the caller `resize()`s to a sensible default; downstream `renderElement` may resize again per `sketchSize`. Auto-layout keeps content aligned during external resize.

For per-edge dividers (list items, header bottom-border), use a 1-px child `figma.createLine()` rather than per-side stroke weights — works on all plugin API versions.

---

## Components

### Button — `button`, `secondary_action`, `destructive_action`, `menu_item`

Full-bleed minus gutter on mobile by default; content-fit on desktop.

```js
function makeButton(el, parentNode, viewport) {
  const isMobile = viewport === "mobile";
  const variant =
    el.type === "destructive_action" ? "destructive" :
    el.type === "secondary_action" || el.type === "menu_item" ? "secondary" :
    "primary";

  const f = autoFrame({
    name: `[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`,
    pad: isMobile ? S[3] : S[2], gap: S[2],
    align: el.type === "menu_item" ? "MIN" : "CENTER",
  });
  f.cornerRadius = R.md;
  f.strokes = [C_INK];
  f.strokeWeight = 1;

  if (variant === "primary") f.fills = [C_FILL_BTN];
  else if (variant === "destructive") { f.fills = []; f.dashPattern = [6, 3]; }
  else f.fills = [];

  const w = isMobile && variant === "primary"
    ? Math.max(parentNode.width - S[5] * 2, 200)
    : (isMobile ? 200 : 120);
  const h = isMobile ? TAP.mobile + 4 : 36;
  f.resize(w, h);

  const label = makeText(el.label || el.type, T.body, "regular");
  label.fills = [C_INK];
  f.appendChild(label);
  return f;
}
```

### Icon button — `icon_button`

Square tap target. Glyph slot drawn as a small placeholder; if `el.label` is 1–2 chars, use it as text instead.

```js
function makeIconButton(el, parentNode, viewport) {
  const size = viewport === "mobile" ? TAP.mobile : TAP.desktop;
  const f = autoFrame({
    name: `[napkin:el:${el.id}] icon_button: ${el.label || ""}`,
    pad: 0, gap: 0, align: "CENTER",
  });
  f.cornerRadius = R.md;
  f.strokes = [C_INK]; f.strokeWeight = 1; f.fills = [];
  f.resize(size, size);

  if (el.label && el.label.length <= 2) {
    const t = makeText(el.label, T.body, "bold");
    t.fills = [C_INK];
    f.appendChild(t);
  } else {
    const glyph = figma.createRectangle();
    glyph.resize(S[4], S[4]);
    glyph.strokes = [C_INK]; glyph.strokeWeight = 1;
    glyph.fills = []; glyph.cornerRadius = R.sm;
    f.appendChild(glyph);
  }
  return f;
}
```

### Input — `text_field`, `textarea`, `search_field`, `select`, `combo_box`

Single-line by default; `textarea` is taller. Search adds a glyph; select adds a chevron caret on the right.

```js
function makeInput(el, parentNode, viewport) {
  const isMobile = viewport === "mobile";
  const isMultiline = el.type === "textarea";
  const isSearch = el.type === "search_field";
  const isSelect = el.type === "select" || el.type === "combo_box";

  const f = autoFrame({
    name: `[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`,
    pad: S[3], gap: S[2], align: "MIN",
  });
  f.counterAxisAlignItems = isMultiline ? "MIN" : "CENTER";
  f.cornerRadius = R.md;
  f.strokes = [C_INK]; f.strokeWeight = 1;
  f.fills = [C_WHITE];

  const w = isMobile ? parentNode.width - S[5] * 2 : 280;
  const h = isMultiline ? (isMobile ? 120 : 96) : (isMobile ? TAP.mobile : 36);
  f.resize(w, h);

  if (isSearch) {
    const ring = figma.createEllipse();
    ring.resize(S[3], S[3]);
    ring.strokes = [C_INK_MUTED]; ring.strokeWeight = 1; ring.fills = [];
    f.appendChild(ring);
  }

  const placeholder = makeText(el.label || el.type, T.body);
  placeholder.fills = [C_INK_MUTED];
  placeholder.layoutGrow = 1;     // take remaining space; pushes caret to right
  f.appendChild(placeholder);

  if (isSelect) {
    const caret = figma.createPolygon();
    caret.pointCount = 3;
    caret.resize(S[2], S[2]);
    caret.rotation = 180;
    caret.fills = [C_INK_MUTED]; caret.strokes = [];
    f.appendChild(caret);
  }
  return f;
}
```

### Toggle — `checkbox`, `radio_group`, `switch`

Indicator + label in a row.

```js
function makeToggle(el, parentNode, viewport) {
  const f = autoFrame({
    name: `[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`,
    pad: 0, gap: S[2], align: "MIN",
  });
  f.fills = [];
  f.resize(Math.min(parentNode.width - S[5] * 2, 280),
           viewport === "mobile" ? TAP.mobile : 28);

  const ind = figma.createRectangle();
  if (el.type === "switch")          { ind.resize(36, 20); ind.cornerRadius = R.full; }
  else if (el.type === "radio_group") { ind.resize(20, 20); ind.cornerRadius = R.full; }
  else                                { ind.resize(20, 20); ind.cornerRadius = R.sm;   }
  ind.strokes = [C_INK]; ind.strokeWeight = 1; ind.fills = [];
  f.appendChild(ind);

  const label = makeText(el.label || el.type, T.body);
  label.fills = [C_INK];
  f.appendChild(label);
  return f;
}
```

### Card — `card`

Padded vertical container; children render inside via `renderElement` recursion.

```js
function makeCard(el, parentNode, viewport) {
  const f = autoFrame({
    name: `[napkin:el:${el.id}] card: ${el.label || ""}`,
    dir: "VERTICAL", pad: S[4], gap: S[3], align: "MIN",
  });
  f.counterAxisAlignItems = "MIN";
  f.cornerRadius = R.lg;
  f.strokes = [C_LINE]; f.strokeWeight = 1;
  f.fills = [C_WHITE];
  const w = Math.min(parentNode.width - S[5] * 2, viewport === "mobile" ? 320 : 360);
  f.resize(w, 160);
  return f;
}
```

### List item — `list_item`

Row with leading label, optional trailing slot, and a thin bottom divider drawn as a child line.

```js
function makeListItem(el, parentNode, viewport) {
  const isMobile = viewport === "mobile";
  const f = autoFrame({
    name: `[napkin:el:${el.id}] list_item: ${el.label || ""}`,
    pad: S[3], gap: S[3], align: "SPACE_BETWEEN",
  });
  f.fills = [];
  f.resize(parentNode.width, isMobile ? 56 : 48);

  const label = makeText(el.label || "Item", T.body);
  label.fills = [C_INK];
  f.appendChild(label);

  // Bottom divider — drawn after, positioned by the parent.
  const div = figma.createLine();
  div.x = 0; div.y = f.height;
  div.resize(f.width, 0);
  div.strokes = [C_LINE]; div.strokeWeight = 1;
  div.name = `[napkin:el:${el.id}:divider]`;
  // The caller appends `div` to the same parent as `f` so the divider sits
  // along the row's bottom edge regardless of auto-layout siblings.
  return { node: f, sibling: div };
}
```

`renderElement` knows to append `sibling` next to `node` when the helper returns `{ node, sibling }`.

### Modal / dialog — `modal`, `dialog`

Centered overlay with a dim scrim.

```js
function makeModal(el, parentNode, viewport) {
  const isMobile = viewport === "mobile";

  // Scrim — sibling on the parent, full-bleed.
  const scrim = figma.createRectangle();
  scrim.name = `[napkin:el:${el.id}:scrim]`;
  scrim.resize(parentNode.width, parentNode.height);
  scrim.fills = [C_SCRIM]; scrim.strokes = [];
  // Caller appends scrim before the modal so z-order is correct.

  const f = autoFrame({
    name: `[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`,
    dir: "VERTICAL", pad: S[5], gap: S[3], align: "MIN",
  });
  f.counterAxisAlignItems = "MIN";
  f.cornerRadius = R.xl;
  f.strokes = [C_INK]; f.strokeWeight = 1;
  f.fills = [C_WHITE];
  const w = isMobile ? Math.min(parentNode.width - S[4] * 2, 320) : 480;
  f.resize(w, 280);

  return { node: f, sibling: scrim };
}
```

`renderElement` centers the modal on the parent after creation; the scrim is appended first so the modal renders on top.

### Alert / toast / state — `alert`, `toast`, `empty_state`, `loading_state`, `error_state`, `confirmation_state`

Inline status with leading dot + message.

```js
function makeAlert(el, parentNode, viewport) {
  const f = autoFrame({
    name: `[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`,
    pad: S[3], gap: S[2], align: "MIN",
  });
  f.cornerRadius = R.md;
  f.strokes = [C_INK]; f.strokeWeight = 1; f.fills = [];
  f.resize(Math.min(parentNode.width - S[5] * 2, 360), 56);

  const dot = figma.createEllipse();
  dot.resize(S[3], S[3]);
  dot.fills = [C_INK]; dot.strokes = [];
  f.appendChild(dot);

  const msg = makeText(el.label || `(${el.type})`, T.body);
  msg.fills = [C_INK];
  f.appendChild(msg);
  return f;
}
```

`error_state` and `loading_state` differ only by label; the visual is the same on the wireframe.

### Badge — `badge`

Pill with short bold text. Hugs content.

```js
function makeBadge(el, parentNode, viewport) {
  const f = autoFrame({
    name: `[napkin:el:${el.id}] badge: ${el.label || ""}`,
    pad: S[2], gap: 0, align: "CENTER",
  });
  f.paddingTop = S[1]; f.paddingBottom = S[1];
  f.cornerRadius = R.full;
  f.strokes = [C_INK]; f.strokeWeight = 1; f.fills = [];
  f.primaryAxisSizingMode = "AUTO";
  f.counterAxisSizingMode = "AUTO";

  const label = makeText(el.label || "badge", T.caption, "bold");
  label.fills = [C_INK];
  f.appendChild(label);
  return f;
}
```

### Avatar — `avatar`

```js
function makeAvatar(el, parentNode, viewport) {
  const e = figma.createEllipse();
  e.name = `[napkin:el:${el.id}] avatar: ${el.label || ""}`;
  e.resize(40, 40);
  e.strokes = [C_INK]; e.strokeWeight = 1; e.fills = [];
  return e;
}
```

### Image placeholder — `image_placeholder`

Rectangle with a single diagonal "X" line — the universal low-fi image cue.

```js
function makeImagePlaceholder(el, parentNode, viewport) {
  const f = figma.createFrame();
  f.name = `[napkin:el:${el.id}] image_placeholder: ${el.label || ""}`;
  const w = 240, h = 160;
  f.resize(w, h);
  f.strokes = [C_INK]; f.strokeWeight = 1; f.fills = []; f.cornerRadius = R.sm;

  const diag = figma.createLine();
  diag.x = 0; diag.y = 0;
  diag.resize(Math.hypot(w, h), 0);
  diag.rotation = -Math.atan2(h, w) * 180 / Math.PI;
  diag.strokes = [C_LINE]; diag.strokeWeight = 1;
  f.appendChild(diag);
  return f;
}
```

### Text — `heading`, `paragraph`, `link`

```js
function makeTextNode(el, parentNode, viewport) {
  const sizes  = { heading: T.heading, h2: T.h2, paragraph: T.body, link: T.body };
  const weight = (el.type === "heading" || el.type === "h2") ? "bold" : "regular";
  const t = makeText(el.label || `(${el.type})`, sizes[el.type] || T.body, weight);
  t.name = `[napkin:el:${el.id}] ${el.type}: ${el.label || ""}`;
  t.fills = [C_INK];
  if (el.type === "link") t.textDecoration = "UNDERLINE";
  return t;
}
```

### Header — `header`

Composed with chrome canonicalization (Recipe 3.5 in `figma-rendering-recipes.md`). Render function defines internal layout only; the canonicalization sets size and edge anchor.

```js
function makeHeader(el, parentNode, viewport) {
  const f = autoFrame({
    name: `[napkin:el:${el.id}] header: ${el.label || ""}`,
    pad: S[4], gap: S[3], align: "SPACE_BETWEEN",
  });
  f.fills = [];
  // Bottom divider drawn as a child line in renderElement after sizing.
  return f;
}
```

### Nav bar — `nav_bar`

Top or bottom; chrome canonicalization decides which. Bottom tab bars distribute children evenly; top nav stacks left.

```js
function makeNavBar(el, parentNode, viewport) {
  const isBottomTab = viewport === "mobile" &&
                      el.sketchPosition && el.sketchPosition.y > 0.85;
  const f = autoFrame({
    name: `[napkin:el:${el.id}] nav_bar: ${el.label || ""}`,
    pad: isBottomTab ? S[2] : S[4],
    gap: S[3],
    align: isBottomTab ? "SPACE_BETWEEN" : "MIN",
  });
  f.fills = [];
  return f;
}
```

### Footer — `footer`

```js
function makeFooter(el, parentNode, viewport) {
  const f = autoFrame({
    name: `[napkin:el:${el.id}] footer: ${el.label || ""}`,
    pad: S[4], gap: S[3], align: "SPACE_BETWEEN",
  });
  f.fills = [];
  return f;
}
```

---

## Custom shape — respect what the user drew

When a sketch mark doesn't map to a taxonomy type — a d-pad, a joystick / orb, a decorative illustration, a custom data viz — emit a `custom_shape` element. The renderer draws the closest primitive faithfully and labels it. **Visual treatment is the same as any regular wireframe element**: solid 1 px ink stroke, no fill, label rendered cleanly below the shape. No dashed strokes, no "?" prefix — those read as errors. A custom shape is intentional content; it should look deliberate.

Use it when:

- The mark is a recognizable shape that's not in the taxonomy (d-pad, joystick, focus rail, trigger button, controller orb).
- The mark is a freeform doodle the user wants preserved (decorative illustration, hand-drawn icon).
- The mark is structured but doesn't fit any taxonomy entry (custom data viz, unique branded shape).

The IR carries `sketchOutline` (`"rect" | "rounded_rect" | "circle" | "ellipse" | "polygon" | "cross" | "line" | "freeform"`) and optional `sketchPoints` (vertex count for polygon). The interpreter picks the closest primitive to what the user drew. Shape size honors `sketchSize`; position honors `sketchPosition` — the literal sketch is the source of truth here.

```js
function makeCustomShape(el, parentNode, viewport) {
  const outline = el.sketchOutline || "rect";
  let shape;

  if (outline === "circle" || outline === "ellipse") {
    shape = figma.createEllipse();
    shape.resize(80, outline === "circle" ? 80 : 60);
  } else if (outline === "polygon") {
    shape = figma.createPolygon();
    shape.pointCount = Math.max(3, el.sketchPoints || 6);
    shape.resize(80, 80);
  } else if (outline === "cross") {
    // Plus / d-pad — composed from two crossed rectangles inside a wrapper.
    shape = figma.createFrame();
    shape.fills = [];
    shape.strokes = [];
    shape.resize(80, 80);
    const arm = 24, span = 80;
    const horiz = figma.createRectangle();
    horiz.resize(span, arm);
    horiz.x = 0; horiz.y = (span - arm) / 2;
    horiz.strokes = [C_INK]; horiz.strokeWeight = 1; horiz.fills = [];
    horiz.cornerRadius = R.sm;
    shape.appendChild(horiz);
    const vert = figma.createRectangle();
    vert.resize(arm, span);
    vert.x = (span - arm) / 2; vert.y = 0;
    vert.strokes = [C_INK]; vert.strokeWeight = 1; vert.fills = [];
    vert.cornerRadius = R.sm;
    shape.appendChild(vert);
  } else if (outline === "line") {
    shape = figma.createLine();
    shape.resize(80, 0);
    shape.strokes = [C_INK]; shape.strokeWeight = 1;
  } else if (outline === "rounded_rect") {
    shape = figma.createRectangle();
    shape.resize(80, 80);
    shape.cornerRadius = R.lg;
  } else {
    // "rect" and "freeform" both render as a rectangle. Freeform is the
    // honest fallback — we can't trace a hand-drawn path in Phase-1, so
    // we draw a clean rect and let the label carry the meaning.
    shape = figma.createRectangle();
    shape.resize(80, 80);
    shape.cornerRadius = R.sm;
  }

  // Common visual treatment — solid ink stroke, no fill, deliberate.
  // Skip for "cross" because its arms set their own strokes above.
  if (outline !== "cross" && outline !== "line") {
    shape.fills = [];
    shape.strokes = [C_INK];
    shape.strokeWeight = 1;
  }

  if (!el.label) {
    shape.name = `[napkin:el:${el.id}] custom_shape`;
    return shape;
  }

  // Wrap the shape with a label below so the wireframe reader sees what it is.
  const wrap = figma.createFrame();
  wrap.name = `[napkin:el:${el.id}] custom_shape: ${el.label}`;
  wrap.fills = []; wrap.strokes = [];
  wrap.layoutMode = "VERTICAL";
  wrap.primaryAxisAlignItems = "CENTER";
  wrap.counterAxisAlignItems = "CENTER";
  wrap.itemSpacing = S[2];
  wrap.paddingTop = 0; wrap.paddingBottom = 0;
  wrap.paddingLeft = 0; wrap.paddingRight = 0;
  wrap.primaryAxisSizingMode = "AUTO";
  wrap.counterAxisSizingMode = "AUTO";

  wrap.appendChild(shape);

  const lbl = makeText(el.label, T.caption);
  lbl.fills = [C_INK];
  wrap.appendChild(lbl);

  return wrap;
}
```

### When to surface in `03 Notes`

Most custom shapes are deliberate (the user knows they drew a d-pad). Don't pollute the Notes page with every one. Add a Notes entry **only** when the model's confidence on what the shape *is* is below 0.6 — that's the signal worth surfacing, not the fact that custom_shape was used. Format:

> Custom mark on screen *Remote Home* — interpreted as "joystick orb" with confidence 0.5. Confirm or correct.

---

## Dispatch

`renderElement` (in `figma-rendering-recipes.md`) maps `el.type` to the right helper. This switch supersedes the legacy archetype switch in Recipe 4.

```js
function createByArchetype(el, parentNode, viewport) {
  switch (el.type) {
    case "button": case "destructive_action": case "secondary_action": case "menu_item":
      return makeButton(el, parentNode, viewport);
    case "icon_button":
      return makeIconButton(el, parentNode, viewport);
    case "text_field": case "textarea": case "search_field":
    case "select": case "combo_box":
      return makeInput(el, parentNode, viewport);
    case "checkbox": case "radio_group": case "switch":
      return makeToggle(el, parentNode, viewport);
    case "card":
      return makeCard(el, parentNode, viewport);
    case "list_item":
      return makeListItem(el, parentNode, viewport);
    case "modal": case "dialog":
      return makeModal(el, parentNode, viewport);
    case "alert": case "toast":
    case "empty_state": case "loading_state":
    case "error_state": case "confirmation_state":
      return makeAlert(el, parentNode, viewport);
    case "badge":
      return makeBadge(el, parentNode, viewport);
    case "avatar":
      return makeAvatar(el, parentNode, viewport);
    case "image_placeholder":
      return makeImagePlaceholder(el, parentNode, viewport);
    case "heading": case "h2": case "paragraph": case "link":
      return makeTextNode(el, parentNode, viewport);
    case "header":
      return makeHeader(el, parentNode, viewport);
    case "nav_bar":
      return makeNavBar(el, parentNode, viewport);
    case "footer":
      return makeFooter(el, parentNode, viewport);
    case "custom_shape":
      return makeCustomShape(el, parentNode, viewport);
    default:
      // Long-tail types fall back to generic container (Phase-2a).
      return makeContainer(el, parentNode.width, Math.min(parentNode.height, 200));
  }
}
```

`makeContainer` and the low-level helpers (`makeFrame`, `makeText`, `ensurePage`) live in `figma-rendering-recipes.md` because they're shared by both orchestration and library code.

A few helpers above return `{ node, sibling }` (list_item divider, modal scrim). `renderElement` handles that shape:

```js
const created = createByArchetype(el, parentNode, viewport);
const node    = created.node    || created;
const sibling = created.sibling || null;
if (sibling) parentNode.appendChild(sibling);
// ...continue with node...
```

---

## Auto-layout caveats

- **`figma.loadFontAsync` first.** Always load Inter Regular and Bold before creating any text or auto-layout frame containing text — auto-layout cannot resize a text node with an unloaded font.
- **`primaryAxisSizingMode`.** `"AUTO"` hugs content; `"FIXED"` lets you `resize()`. The library uses `"FIXED"` by default so screen-level layout (which is absolute) can size containers.
- **`itemSpacing` is the gap between children.** Padding is the gap to the frame edge.
- **Per-edge dividers** are drawn as child `figma.createLine()` rather than per-side stroke weights. Reliable across plugin API versions.
- **Auto-layout + manual `x/y`.** Once a frame is auto-layout, child `x/y` is ignored — children are positioned by the layout engine. To overlap (e.g. a modal over its parent), the overlapping child must be a sibling on the parent canvas, not a child of an auto-layout frame.

---

## What this file does NOT cover

- **Tier-2 components.** ~17 long-tail taxonomy types still use the generic container fallback. They migrate one batch at a time without breaking the dispatch.
- **Component-library inserts.** Phase-1 stance: primitives only. No `figma.importComponentByKeyAsync`.
- **Variants beyond the listed.** Focused, disabled, error, success state variants are Phase-2b.
- **Theming.** Grayscale-only by deliberate constraint. Brand styling enters in Phase-4/5 as headless-style specs only.
