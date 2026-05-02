---
name: judge
description: >-
  Stage 2 of 4 in the RaftLabs Website Prompt System. Use after /build (or
  any initial UI implementation) to critically evaluate the result. Produces
  a structured critique across design, UX, copy, accessibility, and code
  quality. Does NOT fix — only evaluates. Invoke: /judge [optional: file or
  component to judge]
---

# Judge — Stage 2 of 4

This is the **Judge** stage of the RaftLabs 4-stage Website Prompt System:
`/build` → `/judge` → `/feedback` → `/polish`

## What This Stage Does

Delivers an honest, structured critique of the current implementation.
This stage does NOT make any changes — it produces a scorecard that drives Stage 3 (`/feedback`).

The purpose is to see what a sharp, critical eye would find before shipping.

## Input

```
/judge
/judge <file-path or component name>
```

If no argument, look for the most recently created/modified component or page.

## Evaluation Framework

Evaluate across 6 dimensions. For each dimension, give:
- A **score** (1-5 where 5 = excellent)
- Specific **findings** (concrete, actionable — not vague)
- A **priority** for each finding (Critical / High / Medium / Low)

---

### Dimension 1: Visual Design

Score the visual quality as a whole. Ask:
- Does the design have a clear, intentional aesthetic direction?
- Is the typography hierarchy strong and readable?
- Is spacing consistent and intentional? (8px grid? 4px? whatever — is it *consistent*?)
- Is the color usage purposeful or arbitrary?
- Does it look like it was made by someone with taste, or AI-generated generic?

**Common issues to catch:**
- Inconsistent spacing (16px gap here, 17px there)
- Generic sans-serif with no personality
- Too many font weights or sizes
- Shadows and border-radius that don't match across components
- Background/text contrast that barely passes

---

### Dimension 2: User Experience

Score the experience of using this UI. Ask:
- Is the primary action immediately obvious?
- Is the information hierarchy clear — what do you read first?
- Are interactive elements clearly interactive?
- Is there a logical flow from top to bottom?
- What would confuse a first-time user?

**Common issues to catch:**
- CTA button that blends into the background
- Navigation that doesn't communicate the current section
- Form fields without clear labels
- Error states not designed
- No empty state for lists or feeds

---

### Dimension 3: Copy & Content

Score the quality of the text content. Ask:
- Is the headline specific and compelling, or vague and generic?
- Does the copy speak to user benefits, not just features?
- Is the tone consistent throughout?
- Are CTAs action-oriented and clear?
- Is there Lorem ipsum or placeholder text still present?

**Common issues to catch:**
- Headline: "Welcome to Our Platform" (meaningless)
- Generic CTA: "Learn More" (what? why?)
- Feature-first copy ("Built with React") vs benefit-first ("Deploy in minutes")
- Copy that's too long for the section
- Inconsistent voice (formal in one section, casual in another)

---

### Dimension 4: Accessibility

Score against WCAG 2.1 AA as a baseline. Ask:
- Does all text meet 4.5:1 contrast ratio?
- Are interactive elements keyboard-navigable?
- Are all images described?
- Are form inputs labeled?
- Is the focus ring visible?

Use actual contrast ratios when known. Flag specifics (e.g., "gray-400 on white = 2.85:1 — fails AA").

---

### Dimension 5: Responsiveness & Edge Cases

Score how well the implementation handles different contexts. Ask:
- Does it work at 375px mobile?
- At 768px tablet?
- At 1440px wide?
- What happens with very long text?
- What renders before images load?
- What's the loading state for async content?

---

### Dimension 6: Code Quality

Score the implementation quality. Ask:
- Is the component structure clean and composable?
- Are there magic numbers or hardcoded values that should be variables?
- Is there dead code or commented-out sections?
- Are prop types/TypeScript types correct?
- Is there duplication that should be extracted?

---

## Output Format

```
# /judge Report

## Overall Score: X/5

---

### 1. Visual Design — X/5

**Findings:**
| Priority | Finding | Detail |
|----------|---------|--------|
| Critical | [issue] | [specific detail] |
| High     | [issue] | [specific detail] |
| Medium   | [issue] | [specific detail] |

---

### 2. User Experience — X/5
...

### 3. Copy & Content — X/5
...

### 4. Accessibility — X/5
...

### 5. Responsiveness & Edge Cases — X/5
...

### 6. Code Quality — X/5
...

---

## Top 3 Priorities to Fix

1. **[Finding]** — [Why this matters most]
2. **[Finding]** — [Why this matters most]
3. **[Finding]** — [Why this matters most]

---

Stage 2 complete. Run `/feedback "<your notes>"` to provide additional input,
or `/polish` to address the findings above.
```

## Tone

Be direct and honest. A score of 3/5 means "adequate but not good." A score of 5/5 should be rare — it means "this is genuinely excellent and I couldn't improve it."

Do not soften findings with "nice job on X, but...". Just report what you find. The goal is to make the product better, not to make the developer feel good.
