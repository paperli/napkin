# Napkin Test Fixtures

Sketch + expected-IR pairs used to evaluate Napkin's interpretation quality. These act as a regression set: when the Skill changes, run it against these fixtures and compare the produced IR to the expected one.

Phase-1 success criterion: **Napkin produces an IR that round-trips through the schema and is materially equivalent** to `*.expected.json` for at least 80% of fixtures (humans review).

---

## Layout

```
tests/fixtures/
  README.md                  # this file
  <name>/
    sketch.png               # the input sketch (or .jpg / .pdf / .heic)
    context.md               # short text context the user would provide
    expected.json            # the expected Napkin Flow IR (canonical)
    notes.md                 # what makes this fixture interesting; gotchas
```

One folder per fixture keeps inputs and expected output co-located.

---

## How to add a fixture

1. Create a folder under `tests/fixtures/` named after the scenario (`saas_onboarding/`, `ecommerce_checkout/`, `puzzle_game_tv/`, etc.). Use snake_case.
2. Drop the sketch image as `sketch.<ext>`.
3. Write `context.md` — the same one-paragraph context a real user would provide. Keep it short; this is the *minimum* context Napkin should be able to work from.
4. Run Napkin against the fixture by hand. Hand-edit the produced IR until it's what you'd want as the canonical answer. Save as `expected.json`.
5. Write `notes.md` describing what the fixture exercises (ambiguous arrows, missing end state, multi-platform signals, etc.).

---

## Recommended Phase-1 fixture set

Aim for **5–10 fixtures** covering:

| Scenario | What it exercises |
|---|---|
| `saas_onboarding` | linear flow, signup, optional branch, dashboard end state |
| `ecommerce_checkout` | step flow, form fields, confirmation state |
| `dashboard_settings` | hub-and-spoke, sidebar nav, repeated chrome |
| `list_detail` | list → detail navigation, empty state |
| `modal_flow` | modal-on-top, parent screen preserved, dismissal |
| `ambiguous_arrows` | arrow with two plausible targets — should produce open question |
| `missing_end_state` | sketch with no exit — should produce open question |
| `multi_platform_signals` | sketch combining mobile + desktop chrome — should ask for clarification |
| `unreadable_label` | one critical illegible label — should ask, not guess |
| `puzzle_game_tv` *(stretch / forward-looking)* | TV target, voice-primary gameplay screen |

The last fixture isn't expected to render in Phase-1, but its IR should still validate against v0.2, exercising the forward-looking fields.

---

## Evaluating

Until automated diffing exists:

1. Generate the IR via Napkin.
2. Compare to `expected.json` by hand.
3. Material differences (wrong screen count, wrong primary action, missing flow edge) are failures.
4. Cosmetic differences (different label wording, different element order within a screen) are not failures.

Track results in a simple table per fixture (pass / partial / fail) until we build something better.
