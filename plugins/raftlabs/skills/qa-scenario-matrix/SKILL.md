---
name: qa-scenario-matrix
description: Generates a comprehensive QA scenario matrix document for any feature by exploring the codebase, tracing all code paths, state transitions, validation logic, webhooks, and edge cases. Outputs a structured markdown file to docs/<feature>-qa-scenarios.md. Use when you want to systematically document all testable scenarios for a feature before or after implementation — especially for complex flows with multiple states, cancellations, retries, race conditions, or billing logic.
---

# QA Scenario Matrix Generator

Generates a thorough QA scenario matrix for a named feature by deep-reading the codebase and synthesizing every testable path into a structured markdown document.

## Quick Start

```
/qa-scenario-matrix <feature-name> [optional: path hint or description]
```

Examples:
```
/qa-scenario-matrix coupon-redemption
/qa-scenario-matrix appointment-booking amplify/functions/booking/
/qa-scenario-matrix subscription-upgrade "covers plan changes and proration"
```

## Instructions

### Phase 1 — Understand the Feature

Explore the codebase to build a complete mental model:

1. **Locate entry points** — Lambda handlers, API routes, React pages, GraphQL mutations related to the feature.
2. **Trace all code paths** — Follow every branch: happy path, validation failures, retries, async flows, webhooks.
3. **Identify states** — What statuses does the entity move through? (e.g., `active`, `canceled`, `expired`, `incomplete`)
4. **Find validation logic** — What guards/checks exist? What error messages are returned?
5. **Find webhook/event handlers** — What external events (Stripe, GHL, etc.) affect this feature?
6. **Find cron/background jobs** — Any scheduled jobs that change state?
7. **Find race conditions** — Any non-atomic multi-step operations? Any idempotency checks?

### Phase 2 — Design the Groups

Organize scenarios into logical groups. Use whichever apply:

| Group | When to include |
|-------|----------------|
| GROUP A — Main Flow | Core happy-path scenarios (creation, activation, basic use) |
| GROUP B — Abandonment / Re-attempts | Incomplete/orphaned state cleanup and retry behavior |
| GROUP C — Mid-[State] Actions | Actions taken while feature is already active |
| GROUP D — Upgrade / Downgrade | Plan or tier changes while feature is active |
| GROUP E — Renewal / Cycle Behavior | Recurring behavior across billing or time cycles |
| GROUP F — Expiry | Natural expiry, admin deactivation, limit exhaustion |
| GROUP G — Race Conditions | Concurrent requests, webhook deduplication, idempotency |
| GROUP H — Cancellation / Deletion | What happens when the parent entity is removed |

Add, rename, or merge groups as the feature warrants. Every scenario must map to an observable, testable outcome.

### Phase 3 — Write the Matrix

For each scenario row:

| Column | Guidance |
|--------|----------|
| `#` | Prefix + sequential number (e.g., A1, B3) |
| `Scenario` | One sentence: actor + action + relevant condition |
| `Expected Result` | Precise observable outcome — DB state, error message, Stripe event, etc. |
| `Status` | `✅ Handled` / `✅ Fixed in PR` / `⚠️ Discuss` / `❌ Not handled` / `⚠️ Post-deploy hygiene` |

### Phase 4 — Write the Document

Output to `docs/<feature-name>-qa-scenarios.md` using this exact structure:

```markdown
# <Feature Name> — QA Scenario Matrix

> Generated during [context: code review / implementation of X].
> Use this document to validate, discuss, and track coverage before deployment.

---

## <Feature> System Overview

[2–4 sentence description of what the feature does, key concepts, and key states/transitions]

**Key properties / configuration:**

| Property | Effect |
| -------- | ------ |
| ...      | ...    |

---

## GROUP A — [Main Flow Label]

| #  | Scenario | Expected Result | Status |
| -- | -------- | --------------- | ------ |
| A1 | ...      | ...             | ✅ Handled |

---

## GROUP B — [Next Group]

...

---

### Needs Discussion Before Deployment

| # | Issue | Impact |
| - | ----- | ------ |
| X1 | [ambiguous scenario] | [why it matters, what could go wrong] |
```

Only include "Needs Discussion" section if there are genuine open questions — gaps in handling, Stripe/external sync issues, race conditions without locks, etc.

### Phase 5 — Status Labels

Use these consistently:

| Label | Meaning |
|-------|---------|
| `✅ Handled` | Existing code correctly handles this |
| `✅ Fixed in PR` | This PR introduced the fix |
| `⚠️ Discuss` | Behavior is ambiguous or intentionally deferred |
| `⚠️ Post-deploy hygiene` | Non-blocking; minor cleanup needed later |
| `❌ Not handled` | Gap identified — needs work |
| `✅ Intended behavior` | Edge case that is by-design, not a bug |
| `✅ Already handled by frontend` | Validation handled UI-side |

## Guidelines

- Be exhaustive. A scenario matrix is only useful if it covers what developers wouldn't think to test.
- Scenario descriptions must be testable — avoid vague phrases like "invalid input." Say exactly what is invalid.
- Expected results must be observable — a specific error message, DB column value, webhook event, or UI state.
- If you find a real code gap (❌ Not handled), note it clearly but do not fix it — the matrix documents, it does not implement.
- Keep the document self-contained. A QA engineer with no context should be able to read it and know what to test.
- If a group would have fewer than 2 scenarios, merge it into the closest relevant group rather than creating a sparse section.

## Example Output Structure

See `docs/coupon-qa-scenarios.md` in this repo for a complete reference example with 8 groups and ~40 scenarios.
