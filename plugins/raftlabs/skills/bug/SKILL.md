---
name: bug
description: >-
  Universal bug resolution workflow. Supports Asana tasks, user-described bugs,
  URLs to inspect, and error stack traces. Investigates root cause, analyzes
  negative scenarios (edge cases, error states, auth, empty data, race conditions),
  proposes fix, verifies via browser and regression testing, and optionally hands
  off to implementation. Auto-invokes domain-specific skills based on bug
  classification. Invoke: /bug <asana-link-or-gid>, /bug <url>, /bug <description>,
  /bug (interactive)
---

# Universal Bug Resolution Workflow

## Overview

This skill drives a structured workflow for resolving **any** bug — whether reported by QA in Asana, discovered during development, described by a user, or surfaced by an error trace. Unlike `/dev` which starts from pre-written task specs, `/bug` starts from a problem and works backward: understand the issue, classify its domain, investigate root cause, analyze negative scenarios, propose a fix, and only then implement.

**Key principles:**

1. **Don't just fix the symptom.** Find the root cause and validate the fix against product vision.
2. **Don't just follow the happy path.** Every fix must evaluate negative scenarios — error states, empty data, auth failures, race conditions.
3. **Let the bug tell you which skills to use.** Domain classification auto-invokes the right skills (backend, frontend, database, etc.).

**Output:** A well-reasoned fix with negative scenario coverage, optional Asana comment, optional PR on GitHub.

---

## Input Parsing

Parse the `/bug` invocation to determine the entry type and extract relevant context.

### Entry Type Detection (evaluated in order)

| Entry Type                    | Detection                                                                                                                        | Example                                                                                |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **Asana task**                | URL contains `app.asana.com`, OR raw numeric string with 10+ digits                                                              | `/bug https://app.asana.com/0/0/1234567890/f` or `/bug 1234567890`                     |
| **URL to inspect**            | Starts with `http://` or `https://` but is NOT an Asana URL                                                                      | `/bug https://localhost:4001/courses`                                                  |
| **Error trace**               | Contains stack-trace markers: `Error:`, `at `, `.ts:`, `.tsx:`, `Traceback`, or multiline content with file paths + line numbers | `/bug TypeError: Cannot read properties of undefined (reading 'map') at CourseList...` |
| **Free-text description**     | Any other non-empty argument                                                                                                     | `/bug the signup form doesn't show validation errors`                                  |
| **Interactive (no argument)** | Empty `/bug` invocation                                                                                                          | `/bug`                                                                                 |

### Asana GID Extraction (for Asana entry type only)

```
URL pattern: https://app.asana.com/0/<project-or-zero>/<task-gid>[/f]
Regex: /(\d{10,})/g — take the LAST match from the URL (that's the task GID)
If raw numeric string: use as-is
```

### Validation

- If no argument provided → prompt: "Describe the bug you're seeing. Include: what you expected, what happened instead, and steps to reproduce."
- If argument looks like an Asana URL but GID can't be extracted → ask: "Could not parse Asana task ID. Please provide a full Asana URL or numeric GID."

Set `entryType` to one of: `"asana"`, `"url"`, `"error-trace"`, `"description"`, `"interactive"`

---

## Phase 1: Gather Context

**Goal:** Build a complete picture of the issue regardless of how it was reported.

### Branch by entry type:

### 1a. Asana Entry

> **MCP check:** Verify Asana MCP tools are available before proceeding. If `mcp__plugin_asana_asana__*` tools are not listed, tell the user: "Asana MCP is not configured — run `claude plugin update raftlabs` and authenticate via OAuth. Falling back to description-only mode." Then re-classify `entryType` as `"description"` using the Asana task URL as the description text.

1. **Invoke `/asana` skill** — MANDATORY before any Asana MCP call (per project memory)

2. **Fetch task details:**

   ```
   asana_get_task(
     task_id: "<extracted-gid>",
     opt_fields: "name,notes,html_notes,assignee,assignee.name,due_on,projects,projects.name,custom_fields,tags,tags.name,permalink_url,completed,memberships,memberships.section,memberships.section.name"
   )
   ```

3. **Fetch task comments/activity:**

   ```
   asana_get_stories_for_task(
     task_id: "<extracted-gid>",
     opt_fields: "text,type,created_at,created_by.name,resource_subtype",
     limit: 30
   )
   ```

   Filter for `type: "comment"` entries — these contain QA observations, screenshots, reproduction steps.

4. **Present issue summary:**

   ```
   ## Issue: <task name>
   **Asana:** <permalink_url>
   **Assignee:** <assignee name or "Unassigned">
   **Due:** <due_on or "No due date">
   **Section:** <section name from memberships>
   **Tags:** <comma-separated tag names>

   ### Description
   <task notes — cleaned up, not raw HTML>

   ### QA Comments
   <chronological comments with author and date>
   ```

### 1b. URL Entry

> **MCP check:** This step requires Playwright MCP. If `browser_navigate` is not available, skip automated inspection and ask the user to describe what they see at the URL instead, then continue with Phase 2 using the description.

1. **Use Playwright MCP** to inspect the page:
   - `browser_navigate` to the provided URL
   - `browser_snapshot` to capture current DOM state
   - `browser_take_screenshot` for visual evidence
   - `browser_console_messages` to check for JS errors
   - `browser_network_requests` to identify failed API calls

2. **Present findings:**

   ```
   ## Page Inspection: <url>
   **Status:** <loaded / error / blank>
   **Console errors:** <list or "none">
   **Failed network requests:** <list or "none">
   **Visual state:** <brief description of what the page shows>
   ```

3. **Ask user:** "What's wrong with this page? What did you expect to see instead?"

### 1c. Error Trace Entry

1. **Parse the stack trace** to extract:
   - Error type and message (e.g., `TypeError: Cannot read properties of undefined`)
   - File paths and line numbers from the stack frames
   - The originating function/component

2. **Read source files** at the referenced line numbers (use Read tool for each file:line)

3. **Present findings:**

   ```
   ## Error Analysis
   **Error:** <type>: <message>
   **Origin:** <file:line> in <function/component>
   **Stack depth:** <N frames>

   ### Source Context
   <relevant code around the error location>
   ```

4. **Ask user** for reproduction steps if not obvious from the trace.

### 1d. Free-text / Interactive Entry

1. If no argument: ask "Describe the bug you're seeing. Include: what you expected, what happened instead, and steps to reproduce."
2. If free-text provided: acknowledge the description.
3. Ask: "Is there an Asana task for this? (paste link, or 'no')"
   - If yes → switch to 1a flow
   - If no → continue with description only

### After all paths converge

Set `issueContext`:

- `title` — short issue title
- `description` — full description
- `reproductionSteps` — how to reproduce (if known)
- `errorEvidence` — screenshots, console errors, stack traces
- `asanaGid` — if available (null otherwise)
- `affectedUrl` — if available
- `affectedApps` — detected from URLs, file paths, or user input

---

## Phase 2: Classify & Route

**Goal:** Determine the bug's domain so the right skills are auto-invoked.

### Classification Matrix

| Signal                                                                                                   | Classification | Skills to auto-invoke                                     |
| -------------------------------------------------------------------------------------------------------- | -------------- | --------------------------------------------------------- |
| Error in `packages/api/`, `packages/auth/`, `packages/validators/`, `packages/queue/`, `packages/email/` | `backend`      | `/backend`                                                |
| Error in `packages/db/`, constraint violations, migration issues, query errors                           | `database`     | `/database`                                               |
| Error in `apps/web-*/`, `packages/ui/`, React components, CSS/Tailwind, hydration                        | `frontend`     | `/react`, `/next-best-practices`                          |
| Error spans both API + web app layers                                                                    | `fullstack`    | `/backend`, `/database`, `/react`, `/next-best-practices` |
| Error in `apps/mobile/`, `packages/ui-mobile/`                                                           | `mobile`       | (no Playwright, no Next.js tools)                         |
| Auth-related: permissions, roles, sessions, tokens, Better-Auth                                          | `auth`         | `/backend` (auth focus)                                   |

### Auto-classification Logic

1. If stack trace references specific files → classify by file path
2. If URL provided → detect app by port (4001=web-learner, 4002=web-mentor, 4003=web-admin) or path patterns
3. If Asana task → look for tags/section names indicating domain
4. If ambiguous → ask: "This could be a frontend or backend issue. Which should I investigate first, or both?"

Store `bugClassification` and the list of `domainSkills` to invoke.

### Next.js Debug Tools

If classification is `frontend` or `fullstack` and a Next.js web app is involved:

- Use `mcp__next-devtools__nextjs_call` with `get_errors` to fetch current dev server errors
- Use `mcp__next-devtools__nextjs_call` with `get_routes` to understand route structure

### Invoke Domain Skills

Invoke each skill from `domainSkills` now — they load domain-specific patterns and conventions that inform the investigation in Phase 4.

---

## Phase 3: Review Product Documents (Conditional)

**Goal:** Validate the reported issue against the product vision. Determine what SHOULD happen according to specs.

### When to run

- **Always run** if `entryType === "asana"` — QA reports need product validation
- **Ask first** if `entryType !== "asana"`: "Would you like me to check the product documents for expected behavior, or is this clearly a code-level bug?"
- **Skip** if user confirms it's a clear code bug (crash, error, wrong data) — note: "Skipped product document review — this is a code-level bug, not a spec question."

### Steps (when running)

1. **Read the Proposal Document** — invoke `/docx` skill, then read `docs/Proposal Document.docx`
   - This is the original product vision — the "why" behind every feature
   - Look for sections relevant to the reported issue's feature area

2. **Read the PRD** — read `docs/PRD.md` directly (it's markdown, no skill needed)
   - This is the expanded requirements document
   - Find the specific requirements for the affected feature

3. **Cross-reference analysis** — answer these questions:
   - Does the reported behavior contradict the Proposal Document's intent?
   - What does the PRD specify for this feature area?
   - Is there a gap between what was specified and what was built?
   - Is the report actually a bug, or is the current behavior intentional?
   - If a fix was suggested, does that suggestion align with the product vision?

4. **Present findings:**

   ```
   ### Product Document Analysis

   **Proposal Document says:** <relevant excerpt or summary>
   **PRD specifies:** <relevant requirement>
   **Current behavior:** <what's happening now>
   **Expected behavior:** <what should happen per specs>
   **Assessment:** <bug / improvement / working-as-designed / spec gap>
   ```

---

## Phase 4: Investigate the Codebase

**Goal:** Find the root cause and understand all affected areas.

### Steps

1. **Invoke `superpowers:brainstorming`** — explore the problem space, enumerate possible causes, evaluate approaches

2. **Invoke `superpowers:systematic-debugging`** — structured root cause analysis (ALWAYS, not just for obvious bugs):
   - Phase 1: Root Cause Investigation — read errors, reproduce, check recent changes, trace data flow
   - Phase 2: Pattern Analysis — find working examples, compare, identify differences
   - Phase 3: Hypothesis Testing — single hypothesis, test minimally, one variable at a time
   - Phase 4: Implementation — create failing test, single fix, verify

3. **Domain skills are already loaded** from Phase 2 — their patterns and conventions now inform the investigation

4. **Launch investigation** — use Explore agents (via `superpowers:dispatching-parallel-agents` if investigating multiple areas simultaneously):

   **Areas to investigate:**
   - **API routers:** `packages/api/src/routers/` — find the relevant router and procedures
   - **DB schema:** `packages/db/src/schema/` — check table definitions, relations, constraints
   - **Validators:** `packages/validators/src/` — check input validation schemas
   - **Auth:** `packages/auth/src/` — if the issue touches authentication or permissions
   - **Affected apps:** trace the feature across `apps/web-learner/`, `apps/web-mentor/`, `apps/web-admin/`, `apps/mobile/`
   - **Project management docs:** `docs/project-management/` — check if there's a task spec for this feature

5. **Document findings:**

   ```
   ### Investigation Results

   **Root cause:** <clear explanation>
   **Affected files:**
   - <file:line> — <what's wrong>
   - <file:line> — <related code>

   **Affected user flows:**
   - <app> → <flow description>

   **Blast radius:** <what else could break if we change this>
   ```

---

## Phase 5: Negative Scenario Analysis

**Goal:** Don't just fix the reported bug. Proactively identify related failure modes in the same code area.

<HARD-GATE>
NEVER skip this phase. Every bug fix must evaluate negative scenarios.
Even if the reported bug is trivial, the surrounding code may have unhandled failure modes.
</HARD-GATE>

### Mandatory Checklist

For each category, assess relevance to the affected code, then check if it's handled:

### 5.1 Error States

- What happens when the API call fails (500, 503, timeout)?
- What happens when the database query returns an error?
- Is there an `error.tsx` boundary for this route? Does it work?
- Are API errors surfaced to the user or silently swallowed?
- Are oRPC errors properly typed and caught on the client?

### 5.2 Empty/Null/Undefined States

- What if the data is empty (empty array, null object)?
- What if optional fields are missing?
- Are there loading states? What renders before data arrives?
- What if the user has no items yet (first-time user experience)?
- What happens with `noUncheckedIndexedAccess` — are array accesses guarded?

### 5.3 Boundary Conditions

- Maximum length inputs (names, descriptions, URLs)
- Special characters in text fields (quotes, HTML entities, emoji, RTL text)
- Minimum values (0, negative numbers, empty strings)
- Large datasets (pagination, infinite scroll, performance degradation)
- Zod schema boundaries — do validators match DB column constraints?

### 5.4 Auth/Permission Scenarios

- Unauthenticated access — does the redirect work?
- Wrong role (learner accessing mentor page, or vice versa)
- Expired session — does re-auth flow work?
- Permission checks at API level (not just UI hiding) — check `packages/auth/src/permissions.ts`
- Better-Auth session edge cases — cookie expiry, cross-app auth

### 5.5 Concurrency/Race Conditions

- Double-submit (rapid button clicks — is the submit button disabled during request?)
- Navigating away during async operation — does cleanup happen?
- Stale data (another user modified the resource)
- Optimistic updates — what if the server rejects?
- TanStack Query cache invalidation — is it correct after mutations?

### 5.6 Network Failures

- Slow network (loading states, timeouts)
- Offline behavior (graceful degradation?)
- Retry logic — does it exist where needed?
- oRPC error handling on the client side

### Output Format

```
### Negative Scenario Analysis

| Category | Relevant? | Handled? | Finding |
|----------|-----------|----------|---------|
| API failure | Yes | Partial | No error boundary on /courses page |
| Empty data | Yes | No | Crashes when courses array is empty |
| Max length | No | — | No text input in this flow |
| Unauth access | Yes | Yes | Redirect to login works |
| Wrong role | Yes | No | No permission check on this API procedure |
| Double submit | Yes | No | Submit button not disabled during request |
| ... | ... | ... | ... |
```

Any **"No"** or **"Partial"** in the Handled column becomes an **additional fix item** in the proposal (Phase 6).

---

## Phase 6: Propose Solution

**Goal:** Present a well-reasoned fix proposal that covers both the reported bug and negative scenario gaps.

### Steps

1. **Invoke `/code-quality`** — ensure proposed changes follow quality standards (30-line function limit, no magic numbers, DRY, proper naming)

2. **Formulate solution(s):**
   - **Must-fix:** The reported bug — this is the primary deliverable
   - **Should-fix:** Negative scenario gaps found in Phase 5 — prioritized by severity
   - If trade-offs exist → present multiple approaches with pros/cons
   - If QA/user suggested a fix but a better approach exists → present both with reasoning

3. **Explain alignment with product vision** (if Phase 3 was run):
   - How does the proposed fix align with the Proposal Document?
   - Does it satisfy the PRD requirements?
   - Are there any deviations from spec, and if so, why?

4. **Present the proposal:**

   ```
   ### Proposed Fix

   **Must-fix (reported issue):**
   - <what will change and why>

   **Should-fix (negative scenarios):**
   - <gap 1> — <proposed handling>
   - <gap 2> — <proposed handling>

   **Files to modify:**
   - <file> — <change summary>

   **Approach:** <brief description of implementation strategy>
   ```

5. **Ask for user confirmation** using `AskUserQuestion`:
   - Present the proposed approach clearly
   - Ask: "Fix only the reported bug, or also address the negative scenario gaps?"
   - Wait for explicit approval before proceeding

**IMPORTANT:** Do NOT proceed past this phase without user confirmation. The user may want to adjust scope, consult stakeholders, or defer.

---

## Phase 7: Comment on Asana Task (Conditional)

**Goal:** Leave a structured, non-technical comment on the Asana task for QA/PM visibility.

### When to run

- **Only if `asanaGid` is set** — skip entirely for non-Asana bugs
- If skipped, move directly to Phase 8

### Comment Structure

Follow the Asana skill's tone guidelines — write for testers, PMs, and stakeholders, NOT developers.

```
asana_create_task_story(
  task_id: "<gid>",
  text: "<structured comment>"
)
```

### Comment Template

```
Investigation Summary

Issue: <1-sentence user-facing description of the problem>

What's happening: <describe the incorrect behavior from a user perspective>

Root cause: <explain in business/user terms — NO code references, file paths, or technical jargon>

Proposed fix: <what will change for users after the fix>

Affected areas:
- <user flow 1>
- <user flow 2>

<If negative scenario gaps were found and will be addressed:>
Related improvements: During investigation, we also identified additional edge cases that will be addressed:
- <user-facing description of edge case 1>
- <user-facing description of edge case 2>

<If QA suggested something different:>
Note: The suggested approach was <X>. We're taking a different approach because <business reason>.

<If there are limitations or side effects:>
Note: <limitation or side effect in plain language>
```

**CRITICAL:** Re-read the Asana skill's "Comment Tone Guidelines" section. No file paths, function names, hooks, or code internals. No developer jargon. No code snippets. Describe WHAT changed from a user/business perspective, not HOW.

---

## Phase 8: Hand Off to Implementation

**Goal:** Transition from investigation to coding, if the user wants to proceed.

### Steps

1. **Ask the user:** "Investigation complete. Ready to implement the fix?"

2. **If user says yes:**

   **8a. Create implementation plan** — invoke `superpowers:writing-plans`
   - Include the Asana task permalink in the plan's Context section if available (per project memory)
   - Plan should reference findings from Phases 1-5
   - Include negative scenario fixes as separate plan items

   **8b. Use current branch** — implement the fix on whatever branch is currently checked out. Do NOT create a new branch. The user has already chosen the appropriate branch for this work.

   **8c. Implement with TDD** — invoke `/tdd` skill (project-specific):
   - Write failing test(s) for the reported bug first (red-green-refactor)
   - Write tests for negative scenario gaps found in Phase 5 — each unhandled scenario becomes a test case
   - Implement the fix
   - Verify all tests pass
   - Coverage must meet 80% threshold

   **8d. Code quality** — invoke `/code-quality` skill:
   - Ensure implementation follows quality standards
   - Max 30-line functions, no magic numbers, proper naming, DRY

   **8e. Parallel agents** — if fix spans multiple independent areas, invoke `superpowers:subagent-driven-development`

   **8f. Browser verification** (for web app bugs):

   <HARD-GATE>
   NEVER claim a web UI bug is fixed without browser verification.
   Use Playwright MCP to navigate, interact, and screenshot. Console-only verification is insufficient for UI bugs.
   </HARD-GATE>
   - If bug affects `web-learner`, `web-mentor`, or `web-admin`:
     - Navigate to the affected URL via Playwright MCP
     - Reproduce the original bug scenario to confirm it is fixed
     - Test negative scenarios from Phase 5 in the browser:
       - Submit empty/invalid forms
       - Check error state rendering
       - Verify loading states
     - Capture screenshots as evidence
   - If API-only bug: verify via test output
   - If mobile-only bug: skip browser verification (Playwright can't test React Native)

   **8g. Regression guard:**
   - Run tests related to changed files: `pnpm test -- --related <changed-files>`
   - For web UI: browser-navigate to 2-3 adjacent pages and snapshot to confirm they render correctly
   - Report regression results — any failures must be investigated before proceeding

   **8h. Verify before claiming done** — invoke `superpowers:verification-before-completion`
   - Run full verification: `pnpm lint && pnpm check-types && pnpm test`
   - Evidence before claims — never say "fixed" without proof

   **8i. Commit following project conventions:**
   - Emoji prefix + conventional commit format
   - Bug fixes: `🐛 fix(scope): description`
   - Improvements: `✨ feat(scope): description` or `♻️ refactor(scope): description`
   - Use `pnpm commit` for interactive commit formatting

   **8j. Finish branch** — invoke `superpowers:finishing-a-development-branch`
   - Create PR against `development` branch
   - Request code review if appropriate

3. **If user says no:**
   - End gracefully: "Investigation documented. <If Asana: 'The Asana comment has been posted with findings.'> You can re-invoke `/bug` later to pick up implementation."

---

## Skills Auto-Invocation Matrix

| Phase | Skill                                        | When                                              |
| ----- | -------------------------------------------- | ------------------------------------------------- |
| 1     | `/asana`                                     | Asana entry — MANDATORY before any Asana MCP call |
| 1     | Playwright MCP                               | URL entry — inspect the page                      |
| 2     | `/backend`, `/database`                      | Backend/database classification                   |
| 2     | `/react`, `/next-best-practices`             | Frontend classification                           |
| 2     | All of the above                             | Fullstack classification                          |
| 2     | next-devtools MCP                            | Frontend/fullstack bugs in Next.js apps           |
| 3     | `/docx`                                      | When reading Proposal Document                    |
| 4     | `superpowers:brainstorming`                  | Always — explore problem space                    |
| 4     | `superpowers:systematic-debugging`           | Always — structured root cause analysis           |
| 4     | `superpowers:dispatching-parallel-agents`    | When investigating multiple apps/areas            |
| 6     | `/code-quality`                              | Always — validate proposed changes                |
| 8     | `superpowers:writing-plans`                  | Before coding — create implementation plan        |
| 8     | `superpowers:executing-plans`                | Execute the plan with checkpoints                 |
| 8     | `/tdd`                                       | During implementation — project-specific TDD      |
| 8     | `/code-quality`                              | During implementation — quality enforcement       |
| 8     | `superpowers:subagent-driven-development`    | If fix spans independent areas                    |
| 8     | Playwright MCP                               | Web UI verification                               |
| 8     | `superpowers:verification-before-completion` | Before claiming done                              |
| 8     | `superpowers:finishing-a-development-branch` | After implementation — PR/merge                   |
| 8     | `superpowers:requesting-code-review`         | Optional — review completed work                  |

## Other Skill Dependencies

| Skill    | When                           | Why                                      |
| -------- | ------------------------------ | ---------------------------------------- |
| `/asana` | Phase 1a (before any MCP call) | MANDATORY — loads Asana MCP context      |
| `/docx`  | Phase 3 (conditional)          | Read Proposal Document.docx              |
| `/dev`   | Phase 8 (optional)             | If the fix maps to an existing task spec |

---

## Hard Gates

```
<HARD-GATE>
NEVER skip Negative Scenario Analysis (Phase 5). Every bug fix must evaluate edge cases.
Even if the reported bug is trivial, the surrounding code may have unhandled failure modes.
</HARD-GATE>

<HARD-GATE>
NEVER claim a web UI bug is fixed without browser verification (Phase 8f).
Use Playwright MCP to navigate, interact, and screenshot. Console-only verification is insufficient for UI bugs.
</HARD-GATE>

<HARD-GATE>
NEVER invoke Asana MCP tools without first invoking the /asana skill.
</HARD-GATE>

<HARD-GATE>
NEVER run Playwright browser verification on mobile-only bugs.
Playwright MCP controls a desktop browser and cannot test React Native / Expo apps.
</HARD-GATE>
```

---

## Error Handling

| Scenario                                 | Action                                                                                                          |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Asana task not found (404/error)         | Tell user: "Could not find Asana task with GID `<gid>`. Please verify the link."                                |
| Asana task is already completed          | Warn user: "This task is marked as completed in Asana. Proceed anyway?"                                         |
| Proposal Document.docx not found         | Warn: "Could not read Proposal Document. Proceeding with PRD only." Skip docx, continue with PRD.               |
| PRD.md not found                         | Warn: "Could not read PRD. Proceeding with context and codebase investigation only."                            |
| Playwright MCP not available             | Warn: "Browser verification unavailable. Proceeding without visual confirmation." Skip browser-dependent steps. |
| Dev server not running                   | Ask user to start it: "Please run `pnpm dev:<app>` so I can verify in the browser."                             |
| URL returns 404/500                      | Report the HTTP status as part of bug evidence. Continue investigation.                                         |
| Stack trace references non-existent file | Warn: "File referenced in stack trace not found — code may have changed." Ask for updated trace.                |
| No negative scenarios relevant           | Report: "All 6 negative scenario categories assessed — none apply to this specific code path."                  |
| Classification is ambiguous              | Ask user to help classify. Don't guess.                                                                         |
| User declines proposed approach          | Ask what they'd prefer. Re-investigate if needed. Do NOT force any approach.                                    |
| Issue is "working as designed"           | Report finding to user. Suggest updating Asana task with this conclusion. Do NOT implement changes.             |

---

## Key Files (Reference)

| File                               | Purpose                                    |
| ---------------------------------- | ------------------------------------------ |
| `docs/Proposal Document.docx`      | Original product vision — read via /docx   |
| `docs/PRD.md`                      | Expanded requirements — read directly      |
| `docs/project-management/`         | Task specs — consulted for feature context |
| `packages/api/src/routers/`        | API router definitions (oRPC)              |
| `packages/db/src/schema/`          | Database schema (Drizzle ORM)              |
| `packages/validators/src/`         | Zod validation schemas                     |
| `packages/auth/src/`               | Better-Auth config and permissions         |
| `packages/auth/src/permissions.ts` | Role-based permission definitions          |
| `packages/ui/src/components/`      | Shared web UI components (shadcn)          |
| `packages/ui-mobile/`              | Shared mobile components                   |
| `apps/web-learner/`                | Learner web client (port 4001)             |
| `apps/web-mentor/`                 | Mentor web client (port 4002)              |
| `apps/web-admin/`                  | Admin web client (port 4003)               |
| `apps/mobile/`                     | Mobile client (Expo)                       |
