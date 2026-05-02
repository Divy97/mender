---
name: feedback
description: >-
  Stage 3 of 4 in the RaftLabs Website Prompt System. Use to incorporate
  explicit feedback — from stakeholders, user tests, or a /judge report —
  into a concrete implementation plan and execute it. Invoke:
  /feedback "<feedback text>" or /feedback (interactive)
---

# Feedback — Stage 3 of 4

This is the **Feedback** stage of the RaftLabs 4-stage Website Prompt System:
`/build` → `/judge` → `/feedback` → `/polish`

## What This Stage Does

Takes explicit feedback from any source and translates it into concrete, prioritized improvements that are then implemented. The goal is to incorporate real input efficiently — without over-correcting or losing the design direction established in Stage 1.

## Input

```
/feedback "<feedback text>"
/feedback (interactive — will prompt for feedback)
```

Feedback can come from:
- A `/judge` report (paste the top findings)
- Stakeholder review notes ("make the hero bolder", "the CTA is too small")
- User testing observations ("people didn't notice the sign-up button")
- Design review comments
- Your own notes

## Phase 1: Parse and Categorize the Feedback

Read all feedback and categorize each item:

| Category | Description | Examples |
|----------|-------------|---------|
| **Visual** | Design, color, spacing, typography | "Too much white space", "Make the heading bigger" |
| **UX** | Interaction, flow, clarity | "CTA is not obvious", "Form is confusing" |
| **Copy** | Text content, tone, headlines | "The headline is too generic", "Change 'Submit' to 'Get Started'" |
| **Accessibility** | WCAG, contrast, keyboard | "Low contrast on gray text", "Focus ring missing" |
| **Technical** | Code, performance, bugs | "Component re-renders too often", "Image not loading on mobile" |
| **Content** | Missing sections or elements | "We need a testimonials section", "Add pricing comparison table" |

## Phase 2: Resolve Conflicts and Ambiguities

Before implementing, check for:

1. **Conflicting feedback** — e.g., "make it bigger" vs "reduce whitespace". Surface the conflict and recommend a resolution.

2. **Vague requests** — e.g., "make it pop more". Translate into concrete changes: "Increase heading size from 3xl to 5xl, change accent color to higher-contrast orange, add subtle background gradient."

3. **Scope creep** — e.g., "also add a blog section". Flag additions that would significantly expand scope. Ask whether to include them or defer to a follow-up.

Present a **Feedback Translation** before implementing:

```
## Feedback Translation

| Original Feedback | Concrete Change | Priority |
|-------------------|-----------------|----------|
| "hero is too flat" | Add gradient bg + subtle box-shadow on CTA | High |
| "make it feel premium" | Increase letter-spacing on h1, switch to neutral-950 text | Medium |
| "CTA should stand out more" | Change button to filled accent (amber-500), increase size to lg | Critical |
| "add testimonials" | OUT OF SCOPE — defer to next iteration | — |

Conflicts:
- "reduce whitespace" vs "it feels cramped": Recommend keeping padding-y on sections, reducing it only between heading and subheading.

Proceed with these changes?
```

Wait for user confirmation (or proceed if feedback was unambiguous).

## Phase 3: Implement Changes

For each confirmed change:

1. Make the change precisely — don't over-correct or make unrelated adjustments
2. Preserve the original design direction and decisions from Stage 1
3. If a change requires structural refactoring, do it cleanly — don't patch on top of a bad foundation

### Rules
- Only change what the feedback explicitly calls out (or the translation above specifies)
- Do not "improve" things that weren't flagged — that's what `/polish` is for
- If implementing a change would break something else, surface it before changing

## Phase 4: Verify the Changes

After implementing:
1. Re-read the original feedback against the changes made
2. Confirm each item is addressed
3. Note if any item was partially addressed (and why)

## Output Format

```
## Feedback Implementation Complete

### Changes Made

| Feedback Item | Change | Files |
|---------------|--------|-------|
| [item] | [what was done] | [file:line] |
| [item] | [what was done] | [file:line] |

### Deferred Items
- [item] — deferred because [reason]

### Partially Addressed
- [item] — [what was done], [what remains]

---

Stage 3 complete. Run `/polish` for the final refinement pass.
```
