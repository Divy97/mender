---
name: raftkit-release-note
description: >-
  Generates and publishes a release note to Asana from git history. Use when a
  new version has been tagged, after running `pnpm release:staging`, or when the
  user mentions "release note", "changelog", or "what shipped in vX.Y.Z".
  Invoke: /raftkit-release-note [version]
---

# Release Note Generator

## Overview

Automates the creation of a release note as an Asana subtask under "Mentor Staging Release Notes." Reads git history between two version tags, categorizes changes into user-facing summaries, fetches the canonical template and latest example from Asana, formats as Asana-compatible HTML, and creates the subtask via MCP.

**Output:** An Asana subtask with well-formatted release notes and a permalink URL.

---

## Key Constants

| Constant                | Value              | Purpose                                     |
| ----------------------- | ------------------ | ------------------------------------------- |
| Template task GID       | `1213627469874773` | Canonical release note format (fetch first) |
| Master release note GID | `1213635023842588` | Parent of staging + production sections     |
| Staging parent GID      | `1213635023842590` | Subtask parent for staging release notes    |
| Production parent GID   | `1213635023842591` | Subtask parent for production release notes |
| Workspace GID           | `1194107417268910` | RaftLabs workspace                          |
| Project GID             | `1213125604020506` | Mentor v1 project                           |
| GitHub repo             | `Raft-Labs/mentor` | For changelog compare URL                   |

Note: Currently only staging releases are being done, so default parent is the staging GID. Production GID is for future use.

---

## Input Parsing

Parse the `/raftkit-release-note` invocation to determine the target version.

| Pattern                    | Example                         | Behavior                                                    |
| -------------------------- | ------------------------------- | ----------------------------------------------------------- |
| Version argument           | `/raftkit-release-note v0.2.31` | Use `v0.2.31` as current, find previous tag                 |
| Version without `v` prefix | `/raftkit-release-note 0.2.31`  | Normalize to `v0.2.31`                                      |
| No argument                | `/raftkit-release-note`         | Auto-detect latest tag via `git describe --tags --abbrev=0` |

---

## Workflow

### Step 1: Determine Version Range (Asana-first)

**1a. Determine current version:**

```bash
# If version provided, use it. Otherwise:
CURRENT_VERSION=$(git describe --tags --abbrev=0)

# Get release date from tag:
RELEASE_DATE=$(git log -1 --format=%ai "${CURRENT_VERSION}" | cut -d' ' -f1)
```

**1b. Find last release note version from Asana:**

Fetch the most recent subtasks from the staging parent to find the last published release note:

```
Tool: mcp__plugin_asana_asana__asana_get_tasks
Parameters:
  parent: "1213635023842590"
  opt_fields: "name,html_notes,created_at"
  limit: 5
```

Parse the version from the latest release note task name. The naming convention is:
`📋 Release Note — v{VERSION} | {DATE}` → extract the version (e.g., `v0.2.50`).

Use that parsed version as `PREVIOUS_VERSION`.

**Save these fetched subtasks** — they are reused in Step 4 for the formatting example (no duplicate Asana call needed).

**1c. Fallback:** If no release notes exist in Asana yet, or the version cannot be parsed from the task name, fall back to:

```bash
PREVIOUS_VERSION=$(git describe --tags --abbrev=0 "${CURRENT_VERSION}^")
```

**1d. Confirm** the version range with the user before proceeding:

> Generating release note for **v{CURRENT}** (covering changes since last release note: **v{PREVIOUS}**, date: **{DATE}**)

Note: There may be multiple intermediate version tags between `PREVIOUS_VERSION` and `CURRENT_VERSION`. This is expected — the release note should cover ALL changes across that entire range.

### Step 2: Gather Commit Data

Run these in parallel:

1. **Git log** — raw commits between tags:

   ```bash
   git log v{PREVIOUS}...v{CURRENT} --oneline --no-merges
   ```

2. **CHANGELOG.md** — read ALL auto-generated sections between `PREVIOUS_VERSION` and `CURRENT_VERSION` (produced by `commit-and-tag-version`). When multiple intermediate versions exist, include every section in the range. This provides structured categorization by commit type across all intermediate releases.

### Step 3: Fetch the Canonical Template (MANDATORY — do not skip)

Call the Asana MCP to fetch the release note template task:

```
Tool: mcp__plugin_asana_asana__asana_get_task
Parameters:
  task_id: "1213627469874773"
  opt_fields: "name,html_notes"
```

This is the "Release Note Template" task — the **source of truth** for structure, sections, and formatting.

Extract from the response:

- The HTML structure and section order
- Which HTML tags are used (e.g., `<h1>` vs `<h2>`, `<ul>` vs `<ol>`)
- Which sections are template-only and should be omitted (see Step 7)

### Step 4: Use the Latest Release Note as Formatting Example (MANDATORY — do not skip)

**Reuse the subtasks already fetched in Step 1b** — do NOT make a duplicate Asana call.

From the fetched subtasks, identify the most recently created one (skip the template itself). Use this as a concrete example of how the template was last applied — tone, level of detail, and section content.

### Step 5: Analyze and Categorize Changes

Use your judgment to transform commit messages into user-facing descriptions:

- **What's New** — features (`feat` commits). Summarize at user/business level, not commit-message level. Combine related commits into single entries (e.g., 10 commits about "post interactions" = 1 feature entry).
- **Bug Fixes** — `fix` commits. Describe what was broken and what now works.
- **Improvements** — `refactor`, `perf`, `chore` commits that have user-visible impact. Skip purely internal changes (CI, tooling, deps).

Add platform tags to each entry: `(web)`, `(mobile)`, `(web + mobile)`, `(admin)`, `(API)`

Infer platform from commit scopes and file paths:

- `web-learner` → `(web)`
- `web-admin` → `(admin)`
- `web-mentor` → `(mentor web)`
- `mobile` → `(mobile)`
- `api`, `api-pkg` → `(API)`
- `ui`, `validators` → `(web + mobile)` or determine from context
- `db`, `auth` → `(API)` unless clearly user-facing

### Step 6: Generate "What to Test" Section

Derive test scenarios from the features and bug fixes:

- Each item: **Area** — action steps to verify
- Focus on user-facing flows that QA should exercise
- Include both happy path and edge cases where relevant

### Step 7: Compose Asana HTML

Use the HTML structure from the template task fetched in Step 3. Reference the latest example fetched in Step 4 for tone and detail level.

**Omit template-only sections:** "📌 How to Use This Template" and "🔗 Related Tasks" — these exist only in the template, not in actual release notes.

**Follow the template's tag choices exactly** (e.g., `<ul>` vs `<ol>`, `<h1>` vs `<h2>`).

**Allowed HTML tags:** `<body>`, `<h1>`, `<h2>`, `<strong>`, `<em>`, `<s>`, `<u>`, `<code>`, `<a>`, `<ol>`, `<ul>`, `<li>`, `<hr />`

**Formatting reminders (easy to forget):**

- **Compact single-line HTML** — Asana renders whitespace literally. Do NOT pretty-print.
- **NO `<br>`** — use separate `<li>` items or `<hr />` for separation.
- **NO `&amp;`** — use the word "and" instead.
- Wrap everything in `<body>...</body>`

**Section omission rules:**

- If there are no bug fixes, omit the Bug Fixes section entirely (including its `<hr />`)
- If there are no improvements, omit the Improvements section
- Never include empty sections

**Large release handling (100+ commits):** Group aggressively, cap What's New at ~20 items.

### Step 8: Load Asana Skill and Create Subtask

**IMPORTANT:** Load the `/raftkit-asana` skill first before making any Asana MCP calls.

Create the subtask:

```
Tool: mcp__plugin_asana_asana__asana_create_task
Parameters:
  name: "📋 Release Note — v{VERSION} | {YYYY-MM-DD}"
  parent: "1213635023842590"
  html_notes: "{the composed HTML}"
```

### Step 9: Verify and Report

After successful creation:

1. Confirm the subtask was created
2. Return the Asana permalink: `https://app.asana.com/0/0/{NEW_TASK_GID}/f`
3. Show a brief summary of what was included (counts of features, fixes, improvements)

---

## Edge Cases

| Scenario                          | Handling                                                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| No bug fixes in release           | Omit Bug Fixes section entirely                                                                                                       |
| No improvements                   | Omit Improvements section                                                                                                             |
| Hotfix release (1-3 commits)      | Follow same template; sections will be brief                                                                                          |
| Very large release (100+ commits) | Group aggressively, cap at ~20 items per section                                                                                      |
| Version tag not found             | Ask user to confirm the correct tag name                                                                                              |
| Only chore/ci/docs commits        | Note "internal maintenance release" in summary, minimal sections                                                                      |
| Multiple versions since last note | Asana lookup finds last release note version; git log spans full range. CHANGELOG sections for all intermediate versions are included |

---

## Example Output

For a release with 3 features, 2 bug fixes, 1 improvement:

```
Release note for v0.2.31 created successfully!

Asana: https://app.asana.com/0/0/1213716876170401/f

Included:
- 3 features (What's New)
- 2 bug fixes
- 1 improvement
- 6 test scenarios
```
