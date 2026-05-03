---
name: raftkit-verify-story
description: >-
  Cross-check Asana user stories against the codebase to verify all requirements
  are implemented and tested. Fetches the task (including subtasks and comments),
  extracts business requirements, searches the codebase for implementations and
  test coverage, and reports back in PM-friendly language. Fixes gaps using /raftkit-dev
  or /raftkit-bug skills sequentially. Invoke: /raftkit-verify-story <asana-url-or-gid>,
  /raftkit-verify-story (interactive). Use when: "verify story", "cross-check user story",
  "check if task is done", "verify Asana task", "story verification", or when
  given an Asana URL in the context of verifying/cross-checking implementation.
---

# User Story Verification Workflow

## Overview

This skill verifies that an Asana user story is fully implemented in the codebase. It fetches the task with all subtasks and comments, extracts business requirements, checks the code and tests for each requirement, and either confirms alignment or fixes gaps — all while communicating in business language that project managers can read.

**Key principles:**

1. **Comments are requirements too.** Follow-up discussions on Asana tasks refine and extend the original story. Treat every comment with equal priority to the description.
2. **Business language only.** All Asana comments are written for project managers and stakeholders — no file paths, no code references, no developer jargon.
3. **Fix on the current branch.** Never create a new branch. Implement wherever the user currently is.

**Output:** Asana comment confirming alignment, or sequential gap fixes with per-fix Asana updates.

---

## Input Parsing

Parse the `/raftkit-verify-story` invocation to determine the Asana task.

### Entry Type Detection

| Entry Type                    | Detection                      | Example                                                        |
| ----------------------------- | ------------------------------ | -------------------------------------------------------------- |
| **Asana URL**                 | URL contains `app.asana.com`   | `/raftkit-verify-story https://app.asana.com/0/0/1234567890/f` |
| **Raw GID**                   | Numeric string with 10+ digits | `/raftkit-verify-story 1234567890123`                          |
| **Interactive (no argument)** | Empty invocation               | `/raftkit-verify-story`                                        |

### Asana GID Extraction

```
URL pattern: https://app.asana.com/0/<project-or-zero>/<task-gid>[/f]
Regex: /(\d{10,})/g — take the LAST match from the URL (that's the task GID)
If raw numeric string: use as-is
```

### Validation

- If no argument provided → prompt: "Please provide an Asana task URL or GID for the user story you want to verify."
- If argument looks like an Asana URL but GID can't be extracted → ask: "Could not parse Asana task ID. Please provide a full Asana URL or numeric GID."

---

## Phase 1: Data Gathering

**Goal:** Build a complete picture of what the user story requires.

### Step 1 — Invoke `/raftkit-asana` skill

MANDATORY before any Asana MCP call.

### Step 2 — Fetch task details

```
asana_get_task(
  task_id: "<extracted-gid>",
  opt_fields: "name,notes,html_notes,completed,custom_fields,assignee,assignee.name,parent,permalink_url,projects,projects.name"
)
```

Save `permalink_url` — you'll need it for linking back.

### Step 3 — Fetch all comments

```
asana_get_stories_for_task(
  task_id: "<extracted-gid>",
  opt_fields: "text,type,created_at,created_by.name,resource_subtype"
)
```

Filter to `type: "comment"` only — ignore system stories (assignment changes, status updates, etc.).

### Step 4 — Fetch all subtasks (recursive)

```
asana_get_tasks(
  parent: "<extracted-gid>",
  opt_fields: "name,notes,html_notes,completed"
)
```

For each subtask returned:

- Fetch its comments using `asana_get_stories_for_task` (same filter as Step 3)
- If the subtask itself has subtasks, recurse

### Step 5 — Build unified requirements document

Combine everything into a single internal document:

```
## Main Task: [task name]
### Description
[task notes/description]

### Comments (chronological)
- [author] ([date]): [comment text]
- [author] ([date]): [comment text]

## Subtask: [subtask name]
### Description
[subtask notes]
### Comments
- [author] ([date]): [comment text]

## Subtask: [subtask name]
...
```

### Step 6 — Extract discrete requirements

Parse the unified document into a numbered list of distinct, verifiable requirements. Each requirement should be:

- A plain-English statement describing a user-facing behavior or capability
- Specific enough to verify against code (e.g., "Learner can delete their account from settings" not "Account management works")
- Grouped by subtask where natural, otherwise by theme

Display the extracted requirements list in the terminal:

```
## Requirements extracted from: [task name]

1. Learner can delete their account from the settings page
2. Confirmation dialog is shown before account deletion
3. All associated data is removed upon confirmation
4. Email notification is sent after account deletion
5. (Subtask: Subscriptions) Pending subscriptions are cancelled on deletion

Total: 5 requirements
```

---

## Phase 2: Verification

**Goal:** Check each requirement against the codebase and tests.

### Two-layer check per requirement

For each requirement in the list:

**Layer 1 — Code existence:**

- Search the codebase using Grep and Glob for relevant routes, components, functions, DB schema fields, API handlers
- Use Explore agents for deeper tracing when a simple search isn't conclusive
- Verdict: **"implemented"** (found matching code) or **"not found"** (no matching code)

**Layer 2 — Test coverage (only if Layer 1 passes):**

- Look for test files in `__tests__/` directories near the implementation
- Check that the test file covers the specific behavior described in the requirement — not just that a test file exists for the module
- Verdict: **"tested"** (test covers this behavior) or **"untested"** (no test for this specific behavior)

### Classification

| Code exists? | Tests exist? | Classification                              |
| ------------ | ------------ | ------------------------------------------- |
| Yes          | Yes          | Aligned                                     |
| Yes          | No           | Gap — needs tests (`/raftkit-dev`)          |
| No           | No           | Gap — needs implementation (`/raftkit-dev`) |
| Yes          | Buggy/broken | Gap — needs fix (`/raftkit-bug`)            |

### Show progress in terminal

As you verify each requirement, display the result:

```
Verifying requirements...

1. ✅ Learner can delete their account from settings — implemented & tested
2. ✅ Confirmation dialog shown before deletion — implemented & tested
3. ❌ All associated data removed on confirmation — implemented but NO tests
4. ❌ Email notification sent after deletion — NOT FOUND in codebase
5. ✅ Pending subscriptions cancelled on deletion — implemented & tested

Result: 3/5 aligned, 2 gaps found
```

### Overall verdict

- **All aligned** → proceed to Phase 3 (post confirmation comment)
- **Any gaps** → proceed to Phase 4 (gap resolution)

---

## Phase 3: Confirmation Comment (all aligned)

Post a comment to the Asana task confirming all requirements are verified.

```
asana_create_task_story(
  task_id: "<task-gid>",
  text: "<confirmation-comment>"
)
```

### Comment format

The comment lists every requirement that was checked, written in business language. No file paths, no code references, no technical jargon.

**Template:**

```
✅ Story Verified

All requirements from this user story have been cross-checked against the implementation:

• [Requirement 1 in plain English]
• [Requirement 2 in plain English]
• [Requirement 3 in plain English]
...

Everything is aligned with the described requirements. No gaps found.
```

**Done.** The skill ends here when everything is aligned.

---

## Phase 4: Gap Resolution (sequential)

<HARD-GATE>
Do NOT create a new git branch. All fixes happen on the current branch.
</HARD-GATE>

### Step 1 — Present gap list

Display the full gap list in the terminal before starting fixes:

```
## Gaps found in: [task name]

1. ❌ "All associated data removed on confirmation" — implemented but no tests → /raftkit-dev
2. ❌ "Email notification sent after deletion" — not found → /raftkit-dev

Fixing sequentially...
```

### Step 2 — Fix each gap sequentially

For each gap, in order:

1. **Auto-classify the skill:**
   - Code doesn't exist at all → invoke `/raftkit-dev`
   - Code exists but is incomplete, buggy, or missing tests → determine:
     - If the behavior is broken/wrong → invoke `/raftkit-bug`
     - If the behavior is correct but untested or partially missing → invoke `/raftkit-dev`

2. **Invoke the skill** with context about what specifically needs to be done. Pass the requirement as the task description. The skill runs on the current branch — it must NOT create a new branch or PR.

3. **After the fix — post update comment to Asana:**

```
asana_create_task_story(
  task_id: "<task-gid>",
  text: "<update-comment>"
)
```

**Per-gap update comment template:**

```
🔧 Update: Requirement Addressed

"[Requirement in plain English]" — [what was the gap: was missing / needed tests / had an issue] and has now been addressed.

Remaining items being checked: [N] of [total]
```

4. **Move to next gap.**

### Step 3 — Final confirmation comment

After all gaps have been resolved, post a final summary comment:

```
asana_create_task_story(
  task_id: "<task-gid>",
  text: "<final-comment>"
)
```

**Final comment template:**

```
✅ Story Verified — All Gaps Resolved

All requirements have been verified and any gaps have been addressed:

• [Requirement 1] ✓
• [Requirement 2] ✓
• [Requirement 3] — was missing, now implemented ✓
• [Requirement 4] — needed tests, now added ✓
• [Requirement 5] ✓

Everything is now aligned with the described requirements.
```

Requirements that had gaps are called out with what was done, but still in business language. No strikethrough — use a brief note like "was missing, now implemented" to keep it clean.

---

## Key Reference

### Asana MCP Tools Used

| Tool                         | Purpose                        |
| ---------------------------- | ------------------------------ |
| `asana_get_task`             | Fetch task details             |
| `asana_get_stories_for_task` | Fetch comments                 |
| `asana_get_tasks`            | Fetch subtasks (parent filter) |
| `asana_create_task_story`    | Post comments                  |

All tools are prefixed: `mcp__plugin_asana_asana__asana_`

### Skills Invoked

| Skill            | When                                    |
| ---------------- | --------------------------------------- |
| `/raftkit-asana` | Always — before any Asana MCP call      |
| `/raftkit-dev`   | Missing implementation or missing tests |
| `/raftkit-bug`   | Broken or incorrect implementation      |

### Comment Tone Rules

These rules apply to ALL Asana comments posted by this skill:

- ✅ Describe WHAT from a user/business perspective
- ✅ Mention which user flows are affected
- ✅ Keep it concise — bullet points, no walls of text
- ❌ No file paths, function names, or code references
- ❌ No developer jargon or implementation details
- ❌ No code snippets
