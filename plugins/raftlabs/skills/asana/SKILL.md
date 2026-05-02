---
name: asana
description: Use when interacting with Asana for project management, task tracking, or work coordination. MUST be invoked when: user mentions "asana", references an Asana task URL or numeric GID (e.g. 1213639384202046), says "leave a comment", "update the task", "check the task", "mark as done", or any intent to read/write Asana data. Covers creating/updating/deleting tasks, comments, subtasks, dependencies, searching, and managing projects.
---

# Asana MCP Integration

> **MCP check:** Before proceeding, verify that `mcp__plugin_asana_asana__asana_typeahead_search` (or any `mcp__plugin_asana_asana__*` tool) is available.
> If not available: stop and tell the user — "Asana MCP is not configured. It ships with the raftlabs plugin — run `claude plugin update raftlabs` then authenticate via OAuth when prompted. Once done, retry this skill."

## Overview

You have access to 43 Asana MCP tools via the `plugin:asana:asana` server. All tools are prefixed with `mcp__plugin_asana_asana__asana_`. Every Asana object has a GID (globally unique identifier) — use `typeahead_search` to find GIDs before operating on resources.

**Workspace GID:** `# TODO: replace with your Asana workspace GID`
**Project GID:** `# TODO: replace with your Asana project GID`

## Quick Reference

### Core Task Tools

| Tool                 | Purpose                                                                |
| -------------------- | ---------------------------------------------------------------------- |
| `asana_get_task`     | Read task details (use `opt_fields` for full data)                     |
| `asana_create_task`  | Create task directly — supports project, section, html_notes, subtasks |
| `asana_update_task`  | Modify name, notes, assignee, dates, completion, custom fields         |
| `asana_delete_task`  | Delete a task permanently                                              |
| `asana_get_tasks`    | List tasks by project, section, tag, or user task list                 |
| `asana_search_tasks` | Advanced search with 50+ filter options                                |

### Comments & Activity

| Tool                         | Purpose                            |
| ---------------------------- | ---------------------------------- |
| `asana_create_task_story`    | Add a comment to a task            |
| `asana_get_stories_for_task` | Read comments and activity history |

### Dependencies & Subtasks

| Tool                          | Purpose                             |
| ----------------------------- | ----------------------------------- |
| `asana_set_task_dependencies` | Set tasks that this task depends on |
| `asana_set_task_dependents`   | Set tasks that depend on this task  |
| `asana_set_parent_for_task`   | Make a task a subtask of another    |

### Followers

| Tool                          | Purpose                      |
| ----------------------------- | ---------------------------- |
| `asana_add_task_followers`    | Add followers to a task      |
| `asana_remove_task_followers` | Remove followers from a task |

### Search & Discovery

| Tool                     | Purpose                                 |
| ------------------------ | --------------------------------------- |
| `asana_typeahead_search` | Quick search — use FIRST for any lookup |
| `asana_search_tasks`     | Advanced filtered task search           |

### Projects

| Tool                               | Purpose                            |
| ---------------------------------- | ---------------------------------- |
| `asana_get_project`                | Read project details               |
| `asana_get_projects`               | List projects in workspace         |
| `asana_get_projects_for_team`      | List projects for a team           |
| `asana_get_projects_for_workspace` | List projects for workspace        |
| `asana_create_project`             | Create a project directly          |
| `asana_get_project_sections`       | List sections in a project         |
| `asana_get_project_status`         | Get a project status update        |
| `asana_get_project_statuses`       | List all project status updates    |
| `asana_create_project_status`      | Post a project status update       |
| `asana_get_project_task_counts`    | Get task count stats for a project |

### Users & Teams

| Tool                            | Purpose                                |
| ------------------------------- | -------------------------------------- |
| `asana_get_user`                | Get user details — "me", email, or GID |
| `asana_get_workspace_users`     | List all workspace users               |
| `asana_get_team_users`          | List users in a team                   |
| `asana_get_teams_for_workspace` | List teams in workspace                |
| `asana_get_teams_for_user`      | List teams for a user                  |

### Portfolios & Goals

| Tool                                           | Purpose                     |
| ---------------------------------------------- | --------------------------- |
| `asana_get_portfolio` / `asana_get_portfolios` | Read/list portfolios        |
| `asana_get_items_for_portfolio`                | List items in a portfolio   |
| `asana_get_goal` / `asana_get_goals`           | Read/list goals             |
| `asana_create_goal` / `asana_update_goal`      | Create/update goals         |
| `asana_get_parent_goals_for_goal`              | Get parent goals            |
| `asana_update_goal_metric`                     | Update goal progress metric |

### Other

| Tool                                               | Purpose                            |
| -------------------------------------------------- | ---------------------------------- |
| `asana_get_attachment`                             | Get attachment details             |
| `asana_get_attachments_for_object`                 | List attachments on a task/project |
| `asana_get_allocations`                            | Get resource allocations           |
| `asana_get_time_period` / `asana_get_time_periods` | Time period lookups                |
| `asana_list_workspaces`                            | List available workspaces          |

## Task Operations

### Creating Tasks (Single Step)

Tasks are created directly — no preview/confirm needed.

```
asana_create_task(
  name: "Implement user notifications",
  project_id: "1213125604020506",      // creates in this project
  section_id: "1213333716581908",      // optional — specific section
  assignee: "me",                      // or email or GID
  due_on: "2026-03-14",               // YYYY-MM-DD
  start_on: "2026-03-10",             // optional
  html_notes: "<body><h1>Title</h1><ul><li>Requirement 1</li></ul></body>",
  followers: "user1@email.com,user2@email.com"
)
```

**Required:** `name` + one of: `project_id`, `parent` (for subtask), or `workspace + assignee` together.

**Create a subtask:**

```
asana_create_task(
  name: "Subtask name",
  parent: "1213506759986048"    // parent task GID
)
```

### Reading Tasks

```
asana_get_task(
  task_id: "1213506759986048",
  opt_fields: "name,notes,html_notes,completed,assignee,due_on,due_at,projects,custom_fields,tags,dependencies,permalink_url"
)
```

Always include `opt_fields` — without it you only get `gid` and `name`.

### Updating Tasks

```
asana_update_task(
  task_id: "1213506759986048",
  name: "New task name",
  html_notes: "<body>...</body>",   // REPLACES entire description
  assignee: "me",
  completed: true,
  due_on: "2026-03-14",
  custom_fields: "{\"field_gid\": \"value\"}"
)
```

### Deleting Tasks

```
asana_delete_task(task_id: "1213506759986048")
```

### Listing Tasks

```
asana_get_tasks(
  project: "1213125604020506",
  opt_fields: "name,completed,assignee,due_on",
  limit: 100
)
```

Context options (use exactly one): `project`, `section`, `tag`, `user_task_list`

## Rich Text Descriptions

Use `html_notes` on both `asana_create_task` and `asana_update_task`. Content must be wrapped in `<body>` tags.

**CRITICAL: Asana renders whitespace literally.** Do NOT use newlines or indentation between HTML tags — write compact, single-line HTML.

### Supported HTML

```html
<body>
  <h1>Heading</h1>
  <strong>Bold</strong>, <em>Italic</em>, <s>Strike</s>, <u>Underline</u>,
  <code>Code</code>
  <ul>
    <li>Bullet</li>
  </ul>
  <ol>
    <li>Number</li>
  </ol>
  <hr />
</body>
```

### Linking to Asana Objects (@-mentions)

```html
<a data-asana-gid="1213506759986048" />
<a data-asana-gid="1213506759986048" data-asana-dynamic="false">Custom text</a>
```

### Example

```html
<body>
  <h1>User Notifications</h1>
  <strong>Goal:</strong> Add push notifications.
  <h2>Requirements</h2>
  <ul>
    <li>Email notifications</li>
    <li>Push notifications</li>
    <li>In-app center</li>
  </ul>
  <strong>Related:</strong>
  <a data-asana-gid="1213506759986048" data-asana-dynamic="false">Auth task</a>
  <hr />
  <em>Priority: High</em>
</body>
```

## Comments

### Adding a Comment

```
asana_create_task_story(
  task_id: "1213506759986048",
  text: "This is a comment."
)
```

Only use for discussion/feedback — task actions are logged automatically.

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

### Reading Comments & Activity

```
asana_get_stories_for_task(
  task_id: "1213506759986048",
  opt_fields: "text,type,created_at,created_by.name,resource_subtype",
  limit: 20
)
```

Returns comments (`type: "comment"`) and system events (`type: "system"`) chronologically.

## Dependencies & Subtasks

### Set Dependencies

```
asana_set_task_dependencies(
  task_id: "1213506759986048",
  dependencies: "task_gid_1,task_gid_2"
)
```

### Set Dependents

```
asana_set_task_dependents(
  task_id: "1213506759986048",
  dependents: "task_gid_1"
)
```

### Make a Subtask

```
asana_set_parent_for_task(
  task_id: "child_task_gid",
  parent: "parent_task_gid"
)
```

## Search & Discovery

### asana_typeahead_search (Use First)

Quick search across any resource type. Always start here.

```
asana_typeahead_search(
  resource_type: "task",     // task, project, user, team, portfolio, goal, tag, custom_field, project_template
  query: "authentication",
  count: 10
)
```

### asana_search_tasks (Advanced)

Powerful filtered search with 50+ parameters including sort, negation filters, and custom fields.

```
asana_search_tasks(
  text: "authentication",
  assignee_any: "me",
  completed: false,
  projects_any: "1213125604020506",
  due_on_after: "2026-03-01",
  sort_by: "due_date",
  sort_ascending: true,
  opt_fields: "name,completed,assignee,due_on",
  limit: 50
)
```

Key filter categories:

- **Text:** `text`
- **Users:** `assignee_any/not`, `created_by_any/not`, `followers_any/not`, `assigned_by_any/not`
- **Dates:** `due_on/after/before`, `start_on/after/before`, `created_on/after/before`, `completed_on/after/before`, `modified_on/after/before`
- **Context:** `projects_any/all/not`, `sections_any/all/not`, `tags_any/all/not`, `teams_any`, `portfolios_any`
- **Status:** `completed`, `is_blocked`, `is_blocking`, `is_subtask`, `has_attachment`
- **Sort:** `sort_by` (due_date, created_at, completed_at, likes, modified_at) + `sort_ascending`

## Project Operations

### Get Sections

```
asana_get_project_sections(
  project_id: "1213125604020506",
  opt_fields: "name"
)
```

### Create Project

```
asana_create_project(
  name: "Q1 Sprint",
  team: "<team_gid>",           // required for org workspaces
  notes: "Project description",
  default_view: "board",        // list, board, calendar, timeline
  privacy_setting: "private"    // public_to_workspace, private_to_team, private
)
```

### Post Project Status

```
asana_create_project_status(
  project_id: "1213125604020506",
  text: "On track — completed auth module this week.",
  color: "green"               // green, yellow, red
)
```

## User Lookup

```
asana_get_user()                              // defaults to "me"
asana_get_user(user_id: "user@example.com")   // by email
asana_get_user(user_id: "1234567890")         // by GID
```

Prefer `asana_typeahead_search(resource_type: "user", query: "name")` for finding specific users.

## Common Patterns

### Pagination

Use `offset` token from `next_page.offset` in response:

```
asana_get_tasks(project: "...", limit: 100)
asana_get_tasks(project: "...", limit: 100, offset: "<token>")
```

### Date Formats

| Field                 | Format       | Example                |
| --------------------- | ------------ | ---------------------- |
| `due_on` / `start_on` | `YYYY-MM-DD` | `2026-03-14`           |
| `due_at` / `start_at` | ISO 8601     | `2026-03-14T17:00:00Z` |
| Search date filters   | `YYYY-MM-DD` | `2026-03-01`           |

### Workflow: Find → Read → Update

1. `asana_typeahead_search` to find the GID
2. `asana_get_task` with `opt_fields` to read state
3. `asana_update_task` to make changes

## Gotchas & Limitations

### Whitespace in html_notes

Asana renders newlines and indentation literally. Always write compact single-line HTML with no whitespace between tags.

### opt_fields Required for Full Data

`asana_get_task` and `asana_get_tasks` return minimal data by default. Always specify `opt_fields`.

### notes vs html_notes

Both REPLACE the entire description. To append, read existing `html_notes` first, then write back combined content.

### custom_fields is a JSON String

Pass as `"{\"field_gid\": \"value\"}"` — not a JSON object.

### Portfolios May Require Business/Enterprise Plan

Portfolio tools may not work on free/premium Asana plans.

### asana_create_task Requires Context

Must provide one of: `project_id`, `parent`, or both `workspace` + `assignee`.

### Section Placement

To create a task in a specific section, provide both `project_id` and `section_id`. Use `asana_get_project_sections` to find section GIDs first.
