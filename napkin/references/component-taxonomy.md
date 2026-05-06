# Component Taxonomy

The vocabulary Napkin uses to describe screens. When mapping sketch marks to semantic components, draw from this list — do not invent new component types unless the sketch demonstrably needs one.

The taxonomy is **independent of any visual library**. It exists to give the IR consistent terminology across screens and across projects.

---

## Style-less stance

This taxonomy is about *what something is*, not *what it looks like*. Napkin renders everything in low-fidelity grayscale. A `card` and a `dialog` differ in role and behavior, not in pixel detail.

For semantic reference (not styling), Napkin uses **Radix UI** and **React Aria** as benchmarks for web. They are unstyled — exactly the right fit. If a sketch element doesn't have a clean Radix/React-Aria analogue, that's a signal to pause and consider whether it's actually a custom variant of something more standard.

The visual rendering of each type — radius, padding, stroke, internal layout — lives in `wireframe-library.md`. This file defines the **what**; the library defines the **how it draws**.

---

## Phase-1 scope: responsive web

Phase-1 ships only the web vocabulary below. Phase 4 will add mobile-native patterns (sheets, tab bars, safe areas). Phase 5 will add desktop-app, TV, and voice patterns. Per-platform reference files are stubbed in §18 of the PRD but not yet populated.

If a sketch clearly implies a non-web platform (mobile-native nav bar, TV focus rail, voice prompt UI), surface it as an **open question** rather than silently downgrading to web — the user may be on a later-phase target.

---

## Component types

### Layout

Structural containers. Have no behavior; only arrange children.

- `page` — full-screen container, single document scope
- `section` — a labeled region within a page
- `container` — generic bounded region
- `grid` — multi-column layout
- `stack` — vertical or horizontal sequence
- `split_panel` — two regions side-by-side, often resizable
- `sidebar` — secondary panel, usually persistent
- `header` — top-of-page chrome
- `footer` — bottom-of-page chrome

### Navigation

Move the user between screens or sections.

- `nav_bar` — primary horizontal navigation, usually in the header
- `sidebar_nav` — vertical persistent navigation
- `tabs` — peer views with shared parent
- `breadcrumb` — hierarchical location indicator
- `stepper` — linear progress through a multi-step flow
- `pagination` — page-by-page list traversal
- `menu` — clickable menu (overflow, profile, etc.)

### Input

Capture user data.

- `text_field` — single-line text input
- `textarea` — multi-line text input
- `checkbox` — boolean toggle, group-friendly
- `radio_group` — single-choice group
- `select` — dropdown choice
- `combo_box` — input with autocomplete suggestions
- `search_field` — text input with search semantics
- `switch` — boolean toggle, single
- `slider` — numeric range input

### Actions

Do something.

- `button` — primary actionable element
- `icon_button` — actionable element labeled only by icon
- `link` — navigation-styled actionable element
- `menu_item` — entry inside a `menu`
- `destructive_action` — variant of `button` for irreversible operations
- `secondary_action` — variant of `button` for de-emphasized choices

### Content

Display information.

- `card` — bounded content unit, often clickable as a whole
- `list` — sequence of similar items
- `table` — rows and columns of structured data
- `avatar` — user/account image
- `image_placeholder` — image slot, low-fi rendered as a box with a label
- `heading` — structural text
- `paragraph` — body text
- `badge` — small labeled tag (status, count, category)
- `metadata_row` — small row of key-value or attribute pairs

### Feedback

Communicate state or response.

- `alert` — inline persistent message
- `toast` — transient floating message
- `modal` — focused overlay blocking the parent screen
- `dialog` — modal variant for confirmations
- `empty_state` — placeholder when no content exists
- `loading_state` — placeholder while content loads
- `error_state` — placeholder when content fails to load
- `confirmation_state` — success acknowledgment

### Fallback

Escape hatch for marks that don't fit the taxonomy.

- `custom_shape` — a deliberate sketch mark that doesn't fit the taxonomy (d-pad, joystick, focus rail, decorative illustration, custom data viz). Carries `sketchOutline` (`"rect" | "rounded_rect" | "circle" | "ellipse" | "polygon" | "cross" | "line" | "freeform"`) and optional `sketchPoints` (vertex count for polygon). The renderer draws the closest primitive faithfully with a solid 1 px ink stroke and a clean label below — this is the "respect the sketched shape" path. **Not an error placeholder** — visual treatment matches every other wireframe element. Surfaces in `03 Notes` only when the model's confidence on what the shape *is* falls below 0.6. See `wireframe-library.md § Custom shape`.

---

## Canonical layout for chrome

Some types render with canonical layout regardless of how the sketch placed them. The interpreter records only that they exist and what they contain; the renderer snaps them to the canonical anchor and skips `sketchSize`. See `ux-interpretation-principles.md § Canonical layout vs. literal marks` and `figma-rendering-recipes.md § Recipe 3.5`.

| Type | Canonical layout |
|---|---|
| `header` | Full-width, anchored top |
| `nav_bar` (top) | Full-width, anchored top — usually inside `header` |
| `nav_bar` (bottom, mobile/tablet) | Full-width, anchored bottom (mobile tab bar). Detected when the marks sit in the bottom ~15% of the frame |
| `footer` | Full-width, anchored bottom |
| `sidebar`, `sidebar_nav` | Full-height, anchored left |

For non-chrome primitives, mobile imposes tap-target floors (≥ 44 × 44 for buttons; ≥ 44 tall for inputs). Desktop uses smaller archetype defaults.

---

## Mark → component table

A starting reference. Treat as soft guidance — context wins.

| Sketch mark | Likely component type |
|---|---|
| Top horizontal rectangle | `header` containing `nav_bar` |
| Bottom horizontal icon row | `nav_bar` (mobile-style) or `footer` |
| Repeated vertical cards | `list` of `card` |
| Box with label + underline | `text_field` |
| Floating rectangle over content | `modal` or `dialog` |
| Horizontal options at top | `tabs` |
| Numbered horizontal pills | `stepper` |
| Magnifying glass + box | `search_field` |
| Hamburger ≡ | `icon_button` toggling a `menu` or `sidebar_nav` |
| Three dots ⋯ | `icon_button` toggling a `menu` |
| Dashed/empty box | `empty_state` or `image_placeholder` |
| Box with X | dismissable container (often `modal` with close `icon_button`) |
| Two regions side-by-side | `split_panel` |
| Long rectangle near top of screen | `heading` |
| Big primary rectangle with label | `button` |

---

## When to extend the taxonomy

Don't, in Phase-1. If you find yourself wanting to add a new type:

1. Check whether the sketch genuinely shows a *new role*, or just a styled variant of an existing role.
2. Check whether the role is web-specific, or implies a future-phase platform (mobile sheet, TV rail, voice prompt). If it's future-phase, surface as an open question — do not extend.
3. If it really is a missing web role, add an open question to the IR ("This screen contains a component I'd describe as X — does that match your intent?") and document the gap. Don't silently invent.

---

## Naming conventions

When generating IR:

- Component `type` values are **snake_case** and from the lists above.
- Element `id` values are stable across iterations. Use a short readable id (`signup_email_field`), not a UUID.
- Element `label` is the user-facing text from the sketch (or inferred). Preserve user-provided wording when legible.
- Screen `name` is a **short, human-readable title** (`Signup`, `Setup Workspace`). Not a sentence.
- Use the same name for the same component across screens. If the same nav bar appears on three screens, it's the same `nav_bar`, named consistently.
