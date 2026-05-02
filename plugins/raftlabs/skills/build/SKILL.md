---
name: build
description: >-
  Stage 1 of 4 in the RaftLabs Website Prompt System. Use when starting a new
  website, landing page, UI component, or frontend feature from a prompt or
  brief. Generates a complete, high-quality initial implementation. Invoke:
  /build <description or brief>
---

# Build — Stage 1 of 4

This is the **Build** stage of the RaftLabs 4-stage Website Prompt System:
`/build` → `/judge` → `/feedback` → `/polish`

## What This Stage Does

Transforms a description or brief into a complete, working initial implementation.
The goal is a strong first version — not a rough wireframe — that can be meaningfully evaluated in Stage 2 (`/judge`).

## Input

```
/build <description>
```

Examples:
```
/build a SaaS pricing page with three tiers for a project management tool
/build a hero section for an ed-tech startup with a bold CTA
/build an onboarding checklist component for a mobile-first app
```

## Phase 1: Understand the Brief

Read the description carefully and extract:
- **Purpose** — what is this page/component for? What problem does it solve?
- **Audience** — who are the users? What's their context and emotional state?
- **Tone** — professional, playful, bold, minimal, luxury, technical?
- **Constraints** — stack, framework, existing design tokens, or brand colors mentioned

If critical information is missing (e.g., the stack or the primary CTA), ask one focused question. Do not ask for information you can reasonably infer.

## Phase 2: Design Direction

Before writing any code, commit to a clear aesthetic direction:

1. **Choose a concept** — pick ONE direction and name it:
   - Examples: "Dark editorial with accent amber", "Brutalist with oversized type", "Clean SaaS with generous whitespace", "Gradient-rich and modern"
   - The direction should be bold enough to be memorable. Avoid generic.

2. **Define the palette** — 2-3 primary colors, one accent. Justify briefly.

3. **Define typography** — heading and body approach (font family, weight, size hierarchy)

4. **State the one thing** — what is the single visual element or decision that will make this design memorable?

Present this as a brief "Design Direction" summary before writing code:

```
## Design Direction: [Name]

Concept: [One sentence]
Palette: [Primary, Secondary, Accent with hex values]
Typography: [Heading / Body approach]
The memorable element: [What makes this stand out]
```

Get a quick confirmation from the user, or proceed if they said "just build it."

## Phase 3: Implement

Write production-grade code following these rules:

### Code Quality
- Write semantic HTML (correct use of `section`, `article`, `nav`, `header`, `main`, etc.)
- Use Tailwind CSS unless the project has a different styling system
- Components should be self-contained and composable
- No placeholder text left in — use realistic copy that fits the design direction
- No placeholder images — use appropriate CSS gradients, SVG patterns, or `next/image` with public URLs for demo purposes

### UI Excellence
- Every spacing decision should be intentional (use a consistent scale: 4/8/16/24/32/48/64px)
- Hover and focus states on all interactive elements
- Responsive by default — design mobile-first, verify at 375px, 768px, 1280px
- Animations: use subtly (fade-in, slide, scale) — not on every element

### Accessibility
- All images have descriptive `alt` text
- Color contrast meets WCAG AA (4.5:1 for text)
- Interactive elements are keyboard-navigable
- Use `aria-label` where button/link text is ambiguous

### Copy
- Write real, compelling copy — not "Lorem ipsum"
- Headlines should be specific and benefit-oriented
- CTAs should be action-oriented ("Start free trial", not "Click here")

## Phase 4: Output

Deliver:
1. **The complete code** — fully runnable, no TODOs, no placeholders
2. **A brief implementation note** (3-5 sentences) explaining:
   - The design direction chosen and why it fits the brief
   - Any notable technical decisions
   - What to evaluate in Stage 2 (`/judge`)

End with:
```
Stage 1 complete. Run `/judge` to evaluate this implementation critically.
```
