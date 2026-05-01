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
- **< 0.5** — record an `openQuestion` instead of guessing.

Examples:

| Situation | Confidence | Action |
|---|---|---|
| Clear three-card grid with labels | 0.95 | Just render |
| Box that could be a card or a button | 0.6 | Pick one, add assumption |
| Unreadable label, unclear screen role | 0.3 | Open question |

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

## What Napkin must not do during interpretation

- Don't normalize away genuine ambiguity. If the sketch is unclear, the IR should reflect that.
- Don't add elements that aren't in the sketch or in stated context. No "obviously you'd want a search bar here" — that's invention.
- Don't infer business logic. If two arrows leave a button, ask which one fires when.
- Don't pick a platform from a single weak signal. A sketch with a hamburger icon could still be desktop. Look for converging signals.
- Don't collapse two genuinely different screens into one because they look similar. Reused chrome ≠ same screen.
