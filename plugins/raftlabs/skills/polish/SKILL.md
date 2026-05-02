---
name: polish
description: >-
  Stage 4 of 4 in the RaftLabs Website Prompt System. Use as the final
  refinement pass — micro-interactions, animation, accessibility, cross-browser
  edge cases, visual precision, and production readiness. Invoke: /polish
  [optional: specific area to polish]
---

# Polish — Stage 4 of 4

This is the **Polish** stage of the RaftLabs 4-stage Website Prompt System:
`/build` → `/judge` → `/feedback` → `/polish`

## What This Stage Does

This is the final mile. After Build, Judge, and Feedback, the implementation is functionally correct and feedback-addressed. Polish finds and fixes the subtle things that separate "good enough" from "excellent" — the details that users won't articulate but will feel.

## Input

```
/polish
/polish animations
/polish accessibility
/polish <specific area>
```

If a specific area is given, focus polish effort there. Otherwise, run all checks.

## Polish Checklist

Work through each area systematically. Fix what you find. Report what you fixed.

---

### 1. Micro-interactions

The small moments that make UI feel alive:

- [ ] Buttons: hover state changes (not just cursor), active/pressed state, loading state when async
- [ ] Links: underline on hover, color transition (not snap)
- [ ] Form inputs: focus ring, border color transition, placeholder behavior
- [ ] Cards: hover lift (subtle box-shadow or translateY(-2px))
- [ ] Navigation items: smooth underline slide, no harsh snap
- [ ] Toggle/checkbox/radio: smooth transition between states
- [ ] Modals/drawers: fade + scale in, not snap open

**Rule:** Transitions should be 150-250ms. Use `ease-out` for enters, `ease-in` for exits. Never use `linear` for UI elements.

```css
/* Good default */
transition: all 150ms ease-out;

/* For transforms specifically */
transition: transform 200ms ease-out, opacity 150ms ease-out;
```

---

### 2. Animation

Purposeful motion that communicates, not decorates:

- [ ] Hero/headline: fade-in + slight translateY on page load (subtle — 8-16px, 400ms)
- [ ] Content sections: staggered reveal as user scrolls (IntersectionObserver)
- [ ] Success/error states: animate in (not snap)
- [ ] Number counters or progress bars: animate to value
- [ ] Skeleton loaders: pulsing shimmer (not static gray blocks)

**Rule:** Respect `prefers-reduced-motion`. Wrap animations in:
```css
@media (prefers-reduced-motion: no-preference) {
  /* animation code here */
}
```

---

### 3. Typography Refinement

The difference between "readable" and "beautiful":

- [ ] Headings: check `letter-spacing` (large headings often need slight negative tracking: `-0.02em`)
- [ ] Body text: line-height should be 1.5-1.7 for readability
- [ ] Small text (< 14px): check contrast and weight — may need `font-weight: 500` to be readable
- [ ] Numbers: use tabular numbers for prices/stats (`font-variant-numeric: tabular-nums`)
- [ ] Long-form text: check `max-width` (60-75 chars per line is optimal: `max-w-prose` in Tailwind)
- [ ] Balance headings: use CSS `text-wrap: balance` for multi-line headings

---

### 4. Spacing Precision

Inconsistencies that break visual harmony:

- [ ] Verify consistent use of spacing scale (no `17px` or `13px` values — stick to 4/8/12/16/24/32/48/64)
- [ ] Check section padding is consistent across all sections (usually 64-96px top/bottom)
- [ ] Verify component internal padding matches (e.g., all cards have the same internal padding)
- [ ] Check that related elements are visually grouped (closer together) vs unrelated elements (further apart)
- [ ] Verify the visual weight of the page draws the eye in the right direction

---

### 5. Accessibility Final Pass

Meeting the bar, not just checking the box:

- [ ] Run through the entire UI with keyboard only (Tab, Shift-Tab, Enter, Space, Arrow keys)
- [ ] Check all focus rings are visible and styled (not just the browser default blue outline)
- [ ] Verify heading hierarchy is correct (h1 → h2 → h3, no skipping)
- [ ] Check all SVG icons have `aria-hidden="true"` if decorative, or `aria-label` if meaningful
- [ ] Verify form error messages are announced (`role="alert"` or `aria-live="polite"`)
- [ ] Ensure modal/dialog traps focus when open, restores focus on close
- [ ] Check color is not the ONLY indicator of state (use icon or text too)

---

### 6. Responsive Edge Cases

What breaks at the extremes:

- [ ] 320px: Does anything overflow horizontally?
- [ ] 375px: Is tap target size >= 44px for all interactive elements?
- [ ] 768px: Does the layout gracefully transition from mobile to desktop?
- [ ] 1920px: Does content stay contained (max-width wrapper) or stretch uncomfortably?
- [ ] Very long text: Does a 100-character heading break the layout?
- [ ] Very short text: Does a 2-word heading look odd?
- [ ] RTL: If the product will be localized, does the layout work mirrored?

---

### 7. Performance

Checks that affect load time and interaction responsiveness:

- [ ] Images: using `next/image` or `<img loading="lazy">` for below-fold images
- [ ] Hero image: has `priority` prop or `loading="eager"` + `fetchpriority="high"`
- [ ] Fonts: using `font-display: swap` to prevent invisible text during load
- [ ] Icons: SVGs inlined or sprite — not a separate network request per icon
- [ ] Animations: using `transform` and `opacity` only (GPU-accelerated, no layout thrash)
- [ ] No `width: 100%` on images without explicit `height` (causes CLS)

---

### 8. Visual Final Pass

Last visual sanity check before shipping:

- [ ] Take a screenshot and look at it from 1 meter away — does the hierarchy read?
- [ ] Check the design in dark mode if supported
- [ ] Check in Chrome, Safari, Firefox — any rendering differences?
- [ ] Check with browser zoom at 150% — does anything break?
- [ ] Look at the design after a 10-minute break — what catches your eye first? Is that the right thing?

---

## Output Format

```
# /polish Report

## Changes Made

### Micro-interactions
- [change]: [detail]

### Animation
- [change]: [detail]

### Typography
- [change]: [detail]

### Spacing
- [change]: [detail]

### Accessibility
- [change]: [detail]

### Responsive
- [change]: [detail]

### Performance
- [change]: [detail]

## Items Checked and Left Unchanged
- [item]: [why it was already good]

---

The 4-stage Website Prompt System is complete.
Build → Judge → Feedback → Polish ✓
```
