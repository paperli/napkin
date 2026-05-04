# Clarification Questions

When to ask, when to stay quiet, and what good questions look like.

---

## The bar

Ask only when **not asking would materially change the output**.

- "Material" means: a different answer would produce a different IR (different screens, different flow, different platform target).
- "Materially change" rules out cosmetic uncertainty. Don't ask whether a button should be primary or secondary — pick, document as assumption, and move on.

The point is not to be exhaustive. The point is to surface the questions a thoughtful designer would ask before committing.

---

## Triggers (ask when…)

### Screen boundaries are unclear

- Two clusters of UI on the same page: are these separate screens or two states of one screen?
- A grouping that could read as either a sub-component or its own modal.

### Arrows or transitions are ambiguous

- Arrow has no label and could mean click, submit, redirect, or data flow.
- Multiple arrows from the same source — which one is the primary path?
- A loop that could be navigation or could be retry.

### Target platform is unspecified and signals conflict

- Sketch has both desktop chrome (sidebar) and mobile chrome (bottom tab bar).
- User says "responsive" but sketches only mobile.
- Sketch implies TV (focus highlight, rails) but user said "web app."
- Drawn frame ratio and content cues disagree (e.g. ~9:18 portrait frame containing a desktop-style sidebar and dense table).
- Drawn frame ratio is ~1:1 (square) — too ambiguous to default.
- User-named surface conflicts with what every sketch shows (e.g. "mobile_app" but all drawn frames are 16:9).

### Multiple plausible interpretations exist

- A box could be a card, a button, or an empty section.
- A repeated element could be a list or could be three discrete components that just look alike.

### Important labels are unreadable

- Handwriting illegible on a key element.
- Photo too blurry to read a primary CTA.
- Don't guess high-stakes labels — ask.

### Flow is missing start or end states

- No obvious entry point.
- No success / completion screen.
- No error or empty state for a screen that needs one.

### User intent conflicts with the sketch

- User says "checkout flow" but sketch shows a settings page.
- User says "5 screens" but you can only resolve 4.

---

## Triggers (don't ask when…)

### Cosmetic uncertainty

- "Should this card have rounded corners?" → no, Napkin is style-less.
- "Should the button be blue?" → no, grayscale only.

### Missing details that you can reasonably assume

- Single screen labeled "Login" with email + password + button → assume it's a login form, document the assumption.
- Three repeated cards with no labels → assume `list` of `card`, mark confidence ~0.7.
- Drawn frame ratio is unambiguous (≥0.9 confidence on every sketch) **and** the user named a matching surface → infer per-screen `viewport`, record as `assumptions`, proceed without a confirmation turn.

### Things easier to show than ask

- Whether a screen is a modal or a full page → pick the more likely option, render it, and let the user redirect on iteration.

### Things the user already answered

- Don't re-ask context they provided in the prompt.

---

## Question patterns

Group your questions by category and lead with the **most consequential** one. The user should be able to answer in order without re-reading.

### Screen boundaries

- "I see what looks like X screens: [names]. Is the [name] cluster a separate screen, or a state of [other name]?"
- "The bottom-right area looks like it could be its own screen or a modal over [parent]. Which is it?"

### Transitions

- "Arrow from [element] to [screen] — is this a click, a submit, or a redirect after success?"
- "Two arrows leave [button]. Which is the primary path, and what triggers the other?"
- "Is the loop from [screen A] to [screen B] a navigation back, or a retry on failure?"

### Platform / viewport

- "Target surface — responsive web, mobile-native, desktop app, or TV? I'm seeing [signals]."
- "Should I generate desktop-only frames, or desktop + mobile?"
- "Phase-1 of Napkin focuses on responsive web. The sketch implies [other platform] — should I proceed as web for now, or wait?"
- "From the drawn frame ratios I'm reading: [per-sketch breakdown]. Best guess: [surface]. Confirm or correct?"
- "[Sketch] is ~[ratio] — I read that as [viewport]. Want me to treat it as [alt viewport] instead?"
- "You said [surface] but every sketch is ~[ratio], which reads as [other surface]. Which is correct?"

### Optionality / branching

- "Is the [screen] step required for every user, or optional?"
- "Does the flow always end at [screen], or are there alternate end states?"

### Missing states

- "I don't see an empty state for [list/feed]. Should I generate one, or is this list always populated by the time the user gets here?"
- "What's the error state for [submit-action]?"

### Labels

- "I can't read the label on the primary button on screen [N]. Best guess from context: '[guess]'. Confirm or correct?"

### Flow shape

- "I see [N] screens but no obvious entry point. Where does the user start?"
- "Is [screen] the success state, or is there another screen after it?"

---

## A good clarification turn

Mix **stated assumptions** with **necessary questions**. Don't ask without first showing what you've already inferred.

### Example

```
I see 5 screens: Landing, Signup, Setup Workspace, Invite Team, Dashboard.
Reading the arrows, the main path is Landing → Signup → Setup Workspace → Dashboard.
"Invite Team" branches off Setup Workspace.

Working assumptions:
- Landing is the entry point (top-left cluster).
- Dashboard is the success state.
- "Invite Team" is optional.

Two questions before I render:
1. Desktop only, or desktop + mobile?
2. Is the email field on Signup verified inline, or is there a separate verification screen I'm missing?
```

This pattern keeps the conversation efficient: the user can confirm-and-correct in one reply rather than answering twenty atomic questions.

---

## Anti-patterns

Avoid these:

- **Atomic interrogation.** Twenty bullet questions, each a yes/no. Group them.
- **Asking before reading.** Don't ask "what platform?" if the user already said "responsive web" in the prompt.
- **Asking about styling.** Napkin is style-less; styling questions are out of scope.
- **Asking instead of inferring.** If confidence is ≥ 0.8, just proceed and document.
- **Asking instead of marking an open question.** If the user can answer later by editing the IR or the Figma board, save it for `openQuestions` instead of blocking the render.
