---
name: raftkit-asana
description: Use when interacting with Asana for project management, task tracking, or work coordination. MUST be invoked when: user mentions "asana", references an Asana task URL or numeric GID (e.g. 1213639384202046), says "leave a comment", "update the task", "check the task", "mark as done", or any intent to read/write Asana data. Covers creating/updating/deleting tasks, comments, subtasks, dependencies, searching, and managing projects.
---

# Asana MCP Integration

## Overview

Asana MCP tools are available via two servers. **Always prefer Claude's built-in Asana MCP when available; fall back to the V2 proxy server otherwise.**

**Workspace GID:** `1194107417268910` (RaftLabs)
**Mentor v1 Project GID:** `1213125604020506`

### Server Priority

1. **Claude's built-in Asana MCP** (`mcp__claude_ai_Asana__*`) — available only when running inside Claude Code. Use this as the primary server. It is pre-authenticated and does not require any setup.

2. **V2 server via `mcp-remote`** (`asana__*` / `mcp__asana__*`) — works in all other AI tools (Factory/Droid, Codex, OpenCode, Cursor, Windsurf, Kiro, etc.). All configs point to `scripts/asana-mcp.sh` which runs `npx mcp-remote@latest` with proper OAuth client info. Tool names are prefixed by the MCP client — use the tool base name (after the prefix) when referencing them in this skill.

### V2 Proxy Authentication

The V2 server uses OAuth 2.0 with pre-registered client credentials. The `scripts/asana-mcp.sh` wrapper handles the full OAuth flow (browser-based authorization, token refresh). It requires two environment variables:

- `ASANA_CLIENT_ID` — OAuth client ID from the Asana developer console
- `ASANA_CLIENT_SECRET` — OAuth client secret from the Asana developer console

The shell script constructs the `--static-oauth-client-info` JSON blob from these env vars — keeping it out of config files avoids JSON-within-JSON escaping nightmares. Tokens are cached in `~/.mcp-auth/` after the first browser-based authorization.

### V1 Deprecation

The V1 Beta server (`https://mcp.asana.com/sse`) is deprecated and will shut down on **2026-05-11**. The V2 server replaces it entirely.

## Quick Reference

### Core Task Tools

| Tool           | Purpose                                                                          |
| -------------- | -------------------------------------------------------------------------------- |
| `get_task`     | Read task details including comments and subtasks                                |
| `get_tasks`    | List tasks by project, section, tag, or user task list                           |
| `get_my_tasks` | Get tasks assigned to me                                                         |
| `create_tasks` | Create 1–50 tasks in one call; use `parent` for subtasks                         |
| `update_tasks` | Modify 1–50 tasks — name, notes, dates, completion, subtask/dep/follower changes |
| `delete_task`  | Delete a task permanently                                                        |
| `search_tasks` | Advanced search with 50+ filter options                                          |

### Comments & Activity

| Tool          | Purpose                                                                   |
| ------------- | ------------------------------------------------------------------------- |
| `add_comment` | Post a plain-text comment on a task — **`text` param only** (see Gotchas) |

Comments are also returned inline by `get_task` (set `include_comments: false` to suppress).

### Search & Discovery

| Tool             | Purpose                                           |
| ---------------- | ------------------------------------------------- |
| `search_objects` | Quick search across any resource type — use FIRST |
| `search_tasks`   | Advanced filtered task search                     |

### Projects

| Tool                           | Purpose                                                           |
| ------------------------------ | ----------------------------------------------------------------- |
| `get_project`                  | Read project details (pass `include_sections: true` for sections) |
| `get_projects`                 | List projects in workspace                                        |
| `get_status_overview`          | Get project status summary                                        |
| `create_project`               | Create a project (also `create_project_preview` / `_confirm`)     |
| `create_project_status_update` | Post a project status update                                      |

### Users & Teams

| Tool        | Purpose                  |
| ----------- | ------------------------ |
| `get_me`    | Get current user         |
| `get_user`  | Get user by GID or email |
| `get_users` | List workspace users     |
| `get_teams` | List teams in workspace  |

### Portfolios

| Tool                               | Purpose                   |
| ---------------------------------- | ------------------------- |
| `get_portfolio` / `get_portfolios` | Read/list portfolios      |
| `get_items_for_portfolio`          | List items in a portfolio |

### Misc

| Tool              | Purpose                               |
| ----------------- | ------------------------------------- |
| `get_attachments` | List attachments on a task or project |

## Task Operations

### Creating Tasks

`create_tasks` accepts 1–50 tasks and returns `succeeded` / `failed` / `summary`.

**Required per task:** `name` + one of: `project_id`, `parent`, or both `workspace` + `assignee`.

```
create_tasks(
  tasks: [
    {
      name: "Implement user notifications",
      project_id: "1213125604020506",
      section_id: "1213333716581908",     // optional
      assignee: "me",
      due_on: "2026-03-14",              // YYYY-MM-DD
      html_notes: "<body><h1>Title</h1><ul><li>Requirement 1</li></ul></body>",
      followers: "user1@email.com,user2@email.com"
    }
  ]
)
```

**Create a subtask:**

```
create_tasks(
  tasks: [
    {
      name: "Subtask name",
      parent: "1213506759986048"   // parent task GID
    }
  ]
)
```

### Reading Tasks

```
get_task(
  task_id: "1213506759986048",
  include_comments: true,    // default true
  include_subtasks: true,    // default true
  comment_limit: 10          // max 50
)
```

### Reading My Tasks

```
get_my_tasks(
  completed_since: "now",   // "now" = incomplete only; omit for all
  limit: 50
)
```

### Updating Tasks

`update_tasks` accepts 1–50 tasks in one call. Always check `succeeded` and `failed` in the response.

```
update_tasks(
  tasks: [
    {
      task: "1213506759986048",
      name: "New task name",
      html_notes: "<body>...</body>",   // REPLACES entire description
      assignee: "me",
      completed: true,
      due_on: "2026-03-14",
      custom_fields: { "field_gid": "value" }
    }
  ]
)
```

**Subtask / dependency / follower / project changes — all via `update_tasks`:**

```
update_tasks(
  tasks: [
    {
      task: "1213506759986048",
      parent: "parent_task_gid",               // make a subtask (null to promote back)
      add_dependencies: ["dep_gid_1"],
      remove_dependencies: ["dep_gid_2"],
      add_dependents: ["dep_gid_3"],
      add_followers: ["me", "user@email.com"],
      remove_followers: ["other@email.com"],
      add_projects: [{ project_id: "proj_gid", section_id: "sec_gid" }],
      remove_projects: ["proj_gid"]
    }
  ]
)
```

### Deleting Tasks

```
delete_task(task_id: "1213506759986048")
```

### Listing Tasks

```
get_tasks(
  project: "1213125604020506",
  opt_fields: "name,completed,assignee,due_on",
  limit: 100
)
```

Context options (use exactly one): `project`, `section`, `tag`, `user_task_list`

## Rich Text Descriptions

Use `html_notes` on both `create_tasks` and `update_tasks`. Content must be wrapped in `<body>` tags.

**CRITICAL: Asana renders whitespace literally.** Do NOT use newlines or indentation between HTML tags — write compact, single-line HTML.

### Supported HTML

```
<body><h1>Heading</h1><strong>Bold</strong> <em>Italic</em> <s>Strike</s> <u>Underline</u> <code>Code</code><ul><li>Bullet</li></ul><ol><li>Number</li></ol><hr /></body>
```

### Linking to Asana Objects (@-mentions)

```html
<a data-asana-gid="1213506759986048" />
<a data-asana-gid="1213506759986048" data-asana-dynamic="false">Custom text</a>
```

### Example

```
<body><h1>User Notifications</h1><strong>Goal:</strong> Add push notifications.<h2>Requirements</h2><ul><li>Email notifications</li><li>Push notifications</li><li>In-app center</li></ul><strong>Related:</strong> <a data-asana-gid="1213506759986048" data-asana-dynamic="false">Auth task</a><hr /><em>Priority: High</em></body>
```

## Comments

### ⚠️ Use `text` Only — Never `html_text`

The `html_text` parameter is broken in this MCP: Asana stores the content as plain text and the UI renders raw HTML tags literally. Always use `text`.

```
add_comment(
  task_id: "1213506759986048",
  text: "Deployed the auth fix to staging. Login redirect now works correctly for all roles.",
  is_pinned: false   // true to pin at top of activity feed
)
```

**Keep comment text as short prose.** Newline characters may also render as literal `\n` in the UI. If you need structured content, put it in the task description (`html_notes`) and leave a brief comment pointing to it.

### Comment Tone Guidelines

Asana comments are read by testers, project managers, and stakeholders — NOT developers. Write for a non-technical audience.

**DO:**

- Summarize WHAT changed from a user/business perspective
- Mention which user flows are affected (e.g., "login", "checkout", "onboarding")
- Call out anything that needs manual testing or verification
- Note side effects or known limitations in plain language
- Keep it concise — 3-6 bullet points max

**DON'T:**

- Reference file paths, function names, hooks, or code internals
- Use developer jargon (e.g., "databaseHooks", "APIError", "createAuthMiddleware")
- Paste code snippets
- Describe implementation mechanics (HOW it was built)

**Example — BAD (too technical):**

> Added PORTAL_ALLOWED_ROLES config, extracted checkAccountLockout() helper, overrode hooks.before and databaseHooks.session.create.before in createPortalAuth...

**Example — GOOD (business-focused):**

> Fixed: Users can no longer log into apps outside their role.
> • Learners are restricted to the Learner app only
> • Mentors/team members are restricted to the Mentor app only
> • Admins are restricted to the Admin app only
> • Attempting to log in to the wrong app now shows "Invalid email or password"
> • Social login (Google/Apple) is also blocked for wrong-role users
> ⚠️ Note: The mentor application flow is now only accessible from the Mentor app — learners can no longer apply from there. A separate flow will be needed.

### Reading Comments

Comments are returned inline when you call `get_task`. To exclude them:

```
get_task(
  task_id: "1213506759986048",
  include_comments: false
)
```

## Search & Discovery

### search_objects (Use First)

Quick search across any resource type. Always start here to find GIDs.

```
search_objects(
  query: "authentication",
  type: "task"    // task, project, user, team, portfolio, tag
)
```

### search_tasks (Advanced)

Powerful filtered search with 50+ parameters.

```
search_tasks(
  text: "authentication",
  assignee: "me",
  completed: false,
  project_ids: ["1213125604020506"],
  due_on_after: "2026-03-01",
  sort_by: "due_date",
  opt_fields: "name,completed,assignee,due_on",
  limit: 50
)
```

Key filter categories:

- **Text:** `text`
- **Users:** `assignee`, `created_by`, `followers`
- **Dates:** `due_on_after`, `due_on_before`, `created_after`, `modified_after`
- **Context:** `project_ids`, `section_ids`, `tag_ids`
- **Status:** `completed`, `is_subtask`
- **Sort:** `sort_by` (due_date, created_at, modified_at, likes) + `sort_ascending`

## Project Operations

### Get Project (with Sections)

```
get_project(
  project_id: "1213125604020506",
  include_sections: true
)
```

### Create Project

```
create_project(
  name: "Q1 Sprint",
  team_id: "<team_gid>",
  notes: "Project description",
  default_view: "board",          // list, board, calendar, timeline
  privacy_setting: "private"      // public_to_workspace, private_to_team, private
)
```

### Post Project Status

```
create_project_status_update(
  project_id: "1213125604020506",
  text: "On track — completed auth module this week.",
  color: "green"                  // green, yellow, red
)
```

## User Lookup

```
get_me()
get_user(user_id: "user@example.com")   // by email
get_user(user_id: "1234567890")         // by GID
```

Prefer `search_objects(query: "name", type: "user")` for finding specific users.

## Common Patterns

### Pagination

Use `offset` token from `next_page.offset` in response:

```
get_tasks(project: "...", limit: 100)
get_tasks(project: "...", limit: 100, offset: "<token>")
```

### Date Formats

| Field                 | Format       | Example                |
| --------------------- | ------------ | ---------------------- |
| `due_on`              | `YYYY-MM-DD` | `2026-03-14`           |
| `due_at` / `start_at` | ISO 8601     | `2026-03-14T17:00:00Z` |
| Search date filters   | `YYYY-MM-DD` | `2026-03-01`           |

### Workflow: Find → Read → Update

1. `search_objects` to find the GID
2. `get_task` to read current state
3. `update_tasks` to make changes

## Gotchas & Limitations

### ⚠️ `add_comment` `html_text` Is Broken — Always Use `text`

The `html_text` parameter stores content as plain text; the Asana UI renders raw HTML tags literally. Verified in this workspace: do not use `html_text` for comments under any circumstances.

### ⚠️ Comment Newlines Render Literally

Newline characters in comment `text` may appear as literal `\n` in the Asana UI. Keep comments as short prose — no multi-line bullet formatting. Put structured content in `html_notes` on the task instead.

### ⚠️ `start_on` Requires Asana Premium

Setting `start_on` via `update_tasks` returns `{"error":"payment_required"}`. Do not use `start_on`.

### No Comment Deletion

There is no tool to delete comments. Once posted, comments are permanent.

### `update_tasks` / `create_tasks` Are Batch Tools

Always wrap a single operation in a one-element array:

```
update_tasks(tasks: [{ task: "gid", completed: true }])
```

The response always returns `succeeded`, `failed`, and `summary`. Check `failed` even on apparent success — partial failures are possible.

### Whitespace in html_notes

Asana renders newlines and indentation literally inside `html_notes`. Always write compact single-line HTML with no whitespace between tags.

### opt_fields for Listing Tools

`get_task` returns comments and subtasks by default. For listing tools (`get_tasks`, `search_tasks`) you get minimal data unless you specify `opt_fields`.

### notes vs html_notes Both Replace

Both REPLACE the entire description. To append, read existing notes first with `get_task`, then write back the combined content via `update_tasks`.

### Portfolios May Require Business/Enterprise Plan

Portfolio tools may not work on free/premium Asana plans.
