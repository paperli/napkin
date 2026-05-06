# UX Interpretation Principles

How Napkin should *look at* a sketch. These principles drive the interpretation step (Step 2 of the workflow).

---

## Core thesis

**Transcribe intent, not marks.**

A rectangle in a sketch is not "a rectangle." It is a header, a card, a modal, an input — depending on its role in the flow. Napkin's job is to recover that role and represent it semantically.

Tracing the sketch literally would just produce a worse copy of the original. Recovering the intent produces a usable design artifact.

---

## What to identify

When reading a sketch, work through these in order:

### 1. Screen boundaries

Where does one screen end and the next begin? Signals:

- explicit frame outlines on paper
- arrows that originate from one cluster and land in another
- a numbered or labeled grouping ("1", "2", "step 1")
- repeated UI structure (nav bar redrawn) suggesting separate screens vs. shared chrome

If two clusters share a nav bar but differ in body content, they are usually two screens of the same app, not one screen with two states.

### 2. The user's journey

What is the user trying to accomplish from start to end? Identify:

- the **entry point** (where the user lands first)
- the **exit point** (success state, end of flow)
- the **primary action** on each screen (the one verb the screen is about)
- **decision points** (branches that produce different paths)

If you can't name the entry and exit, the flow is incomplete — either ask, or surface as an open question.

### 3. Navigation model

How does the user move between screens? Common patterns:

- linear step flow (signup, checkout, onboarding)
- hub-and-spoke (dashboard with detail pages)
- tabbed (multiple peer views with shared chrome)
- modal-on-top (overlay over a parent screen)
- back-stack (mobile-style push/pop navigation)

The arrows in the sketch encode this. If the arrow shape is ambiguous (does it mean "click" or "data flows here"?), ask or mark assumption.

### 4. UI hierarchy within a screen

For each screen, what is the **primary** content vs. **secondary** chrome? Look for:

- size and position (top + center are usually primary)
- repetition (a row of identical shapes is a list)
- labels (the labeled element is usually the actionable one)
- visual emphasis (boxes, underlines, asterisks in the sketch)

### 5. Repeated patterns

If the same shape recurs across screens, treat it as a **shared component**. Examples:

- top rectangle present on every screen → app header / nav bar
- bottom row of three icons on every screen → tab bar
- recurring 3-card grid → standardized card list

This matters for the design-system mindset. Repeated patterns should map to the same component type with the same name across screens.

### 6. Missing states

Sketches often capture only the happy path. Real flows need:

- empty state (no content yet)
- loading state
- error state
- success / confirmation state

If a sketch has a list, ask: what does this look like before there's anything in it? If the answer isn't in the sketch, surface as an open question — do not invent.

### 7. Information architecture

For multi-screen flows, ask: does the IA make sense?

- Are settings nested under the right parent?
- Does the back button go where you'd expect?
- Is the user always one click from the primary action?

If the IA in the sketch is confusing, capture that as an open question — Napkin's job is to surface confusion, not silently "fix" it.

---

## Confidence handling

Every screen and element in the IR carries a `confidence: number` field (0.0–1.0).

- **≥ 0.8** — proceed silently.
- **0.5–0.8** — proceed but record an entry in `assumptions`.
- **< 0.5** — either record an `openQuestion` (when the ambiguity *blocks* rendering — entry point unclear, arrow direction reversed) or emit a `custom_shape` element (when the mark exists but doesn't fit the taxonomy — a d-pad, joystick, focus rail, decorative illustration, custom data viz). `custom_shape` is rendered as a clean primitive that respects what the user drew (solid stroke, label below) — *not* as a dashed/uncertain placeholder. See `component-taxonomy.md § Fallback` and `wireframe-library.md § Custom shape`.

When emitting `custom_shape`, pick the `sketchOutline` that best matches the user's mark, and supply `sketchPoints` for polygons:

| User drew | `sketchOutline` | `sketchPoints` |
|---|---|---|
| Circle / orb / joystick ball | `circle` | — |
| Oval / capsule | `ellipse` | — |
| Triangle | `polygon` | 3 |
| Pentagon | `polygon` | 5 |
| Hexagon | `polygon` | 6 |
| Octagon | `polygon` | 8 |
| D-pad / plus / cross | `cross` | — |
| Rounded rectangle (badge-like custom) | `rounded_rect` | — |
| Sharp rectangle | `rect` | — |
| Single line / divider / arrow shaft | `line` | — |
| Hand-drawn cloud, lightning, anything we can't classify as a primitive | `freeform` | — |

The label on a `custom_shape` is the user's name for the element ("D-pad", "Joystick", "Logo cloud") — not "(?)" or "(unknown)". The shape is intentional; treat it as such.

Examples:

| Situation | Confidence | Action |
|---|---|---|
| Clear three-card grid with labels | 0.95 | Just render |
| Box that could be a card or a button | 0.6 | Pick one, add assumption |
| Unreadable label, unclear screen role | 0.3 | Open question |
| Freeform doodle (cloud, lightning, custom illustration) | 0.3 on type | `custom_shape` honoring the drawn outline |

Do not silently merge ambiguous sketches into a clean output. The Figma board should *show* uncertainty — it's part of the artifact's value.

---

## The "rough mark → semantic" lens

Napkin should normalize marks into semantic interpretations. A non-exhaustive map:

| Sketch mark | Likely semantic |
|---|---|
| Top horizontal rectangle | Header / nav bar |
| Bottom horizontal row of icons | Tab bar (mobile) or footer nav |
| Vertical column of repeated cards | Card list / feed |
| Box with a label and an underline | Text input |
| Floating rectangle covering part of a screen | Modal or popover |
| Horizontal row of options at the top | Tabs or segmented control |
| Arrow from a button to a different screen | Click transition |
| Dashed box | Empty state / placeholder |
| Box with an "X" or "✕" | Closeable / dismissable element |
| Stack of slightly offset rectangles | List items or stacked cards |
| Magnifying glass icon | Search field |
| Hamburger ≡ icon | Menu toggle |
| Three dots ⋯ | Overflow menu |

This is a **starting point**, not a rulebook. Context overrides the table — a "modal" in a sketch is only a modal if its parent screen is recognizable behind it.

The full taxonomy of valid component types is in `component-taxonomy.md`.

---

## Canonical layout vs. literal marks

Once you've named the semantic role, promote chrome and primary-action elements to **canonical UX layout**. The sketch tells you that they exist and what's in them — not their exact size or pixel coordinates.

A bottom tab bar in a sketch is often drawn as three small circles in a corner. Literal interpretation produces three small circles in a corner. Canonical interpretation produces one full-width `nav_bar` anchored to the bottom, containing three `icon_button` children evenly distributed. The canonical answer is what the user actually meant.

### Group rough marks into chrome containers

When you see these patterns, group the marks into a single semantic chrome container with children — do not leave them as loose top-level elements:

| Pattern | Group as |
|---|---|
| 2–5 peer items in a horizontal row at the bottom of a mobile/tablet screen — *any* mark style: icons, dots, circles, labeled rectangles, text labels | One `nav_bar` (bottom tab bar) with `icon_button` children |
| Logo + links + auth buttons across the top | One `header` containing `nav_bar` with `link` / `button` children |
| Back/title/action marks across the top of a mobile screen | One `header` with `icon_button` and `heading` children |
| Vertical icon column on the left of a desktop screen | One `sidebar` containing `sidebar_nav` with children |

**Group even when each mark looks like a button.** This is the most common interpretation failure: the user draws three labeled rectangles at the bottom of a mobile screen ("Home", "Search", "Profile"), and the literal reading is "three buttons." The semantic reading is one tab bar. The clue is: **2–5 peer items on the bottom edge of a mobile/tablet screen, evenly spaced** — that's almost always a bottom tab bar regardless of mark style. Use `nav_bar` with `icon_button` children even if the user wrote text labels (the labels become the icon-button labels).

Children's `sketchPosition` is normalized to the chrome container, not the screen (see `napkin-flow-ir-schema.md`).

### Skip sketchSize for chrome; use anchor for sketchPosition

For these types, **do not record `sketchSize`** on the IR — the renderer applies canonical dimensions. Record `sketchPosition` only as the anchor cue (e.g. `y: 0.95` for a bottom nav, `y: 0.05` for a top header):

| Type | Canonical anchor |
|---|---|
| `header` | top, full-width |
| `nav_bar` (top) | top, full-width — usually inside `header` |
| `nav_bar` (bottom, mobile/tablet) | bottom, full-width (a.k.a. tab bar). Trigger: viewport is `mobile` or `tablet` and the marks sit in the bottom ~15% of the frame |
| `footer` | bottom, full-width |
| `sidebar`, `sidebar_nav` | left, full-height |

### Canonical sizing for primary primitives

Mobile primitives have tap-target conventions that override rough size from the drawing:

- **Mobile buttons / inputs**: ≥ 44 tall; primary CTAs are typically full-bleed minus a 16 px gutter.
- **Desktop buttons / inputs**: archetype defaults (smaller) are correct.

If the user clearly drew a small secondary action (a text link, an icon-only button), honor that and record `sketchSize`. The override applies to the primary path.

### Why this matters

A sketch carries two kinds of signal: *what's here* (highest fidelity) and *roughly where it is* (lower fidelity). Honoring rough size and position equally produces a wireframe that looks like a hand-tremor copy of the original. Canonical layout for chrome + viewport-aware sizing for primitives lets the wireframe read as a real low-fi UI while still preserving structural intent.

---

## What Napkin must not do during interpretation

- Don't normalize away genuine ambiguity. If the sketch is unclear, the IR should reflect that.
- Don't add elements that aren't in the sketch or in stated context. No "obviously you'd want a search bar here" — that's invention.
- Don't infer business logic. If two arrows leave a button, ask which one fires when.
- Don't pick a platform from a single weak signal. A sketch with a hamburger icon could still be desktop. Look for converging signals.
- Don't collapse two genuinely different screens into one because they look similar. Reused chrome ≠ same screen.
