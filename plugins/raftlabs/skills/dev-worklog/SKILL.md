---
name: dev-worklog
description: Create grounded developer work updates from local repo activity. Use when user wants an end-of-day update, standup note, Jira/Asana comment, weekly summary, or concise worklog based on what changed today.
---

# Dev Worklog

Turn local engineering activity into a concise status update.

Do not invent work. Only summarize what can be grounded in local evidence.

## Use Cases

- end-of-day update
- standup update
- Jira comment draft
- Asana update draft
- weekly work summary
- "what did I do today?"

## Default Sources

Start with local repo evidence:

1. branch name
2. current status
3. commits from requested time window
4. changed files
5. diff summary
6. ticket refs in branch names or commit messages
7. PR refs or remote repo context when available

Good default commands:

```bash
git branch --show-current
git status --short
git log --since="today" --stat --oneline --decorate
git diff --stat
git diff --name-only
git remote -v
```

If the user gives a different time window, use that instead of `today`.

If the repo is not a git repo, say so clearly and ask for another source of truth.

## Workflow

1. Identify output type:
   - standup
   - end-of-day update
   - Jira/Asana comment
   - weekly summary
2. Gather local evidence with git first.
3. Infer:
   - what was completed
   - what is still in progress
   - what changed materially
   - what should be mentioned as validation, deploy, review, or follow-up
   - which Asana ticket links or PR links can be attached
4. Draft concise output in the user's tone.
5. Separate facts from inference.

## Output Rules

- Be concise.
- Prefer bullets.
- Mention concrete work, not vague activity.
- Use file/module names only if they help.
- Do not claim deploys/tests/reviews unless supported by evidence or user context.
- Include Asana ticket links when they can be grounded from local evidence or user-provided context.
- Include PR links when they can be grounded from local evidence or user-provided context.
- If a ticket id or PR number is visible but the full URL cannot be derived safely, mention the ref without inventing the link.
- If evidence is thin, say that and keep output modest.
- If user asks for a ticket comment, write in ready-to-paste form.

## Heuristics

Map evidence to useful update language:

- commits + changed files -> completed implementation work
- dirty working tree -> in-progress work
- test file changes -> testing likely happened
- config/deploy file changes -> possible release/deploy prep
- docs changes -> documentation/update work
- branch names / commit text with ticket ids -> possible Asana/Jira refs
- commit decorations / merge commits / branch refs -> possible PR refs

Be careful:

- changed code does not always mean finished
- commit message quality may be poor
- local repo does not prove Jira/Asana status changes
- do not fabricate URLs from partial ids unless the base URL is known

## Link Detection

Look for ticket and PR refs in:

- branch names
- commit subjects and bodies
- merge commit text
- remote origin URL
- pasted user context

Ticket links:

- If the user has given an Asana workspace or project URL pattern in the thread, use it.
- If full Asana URLs already appear in branch names, commit text, or pasted notes, preserve them.
- If only an internal ticket token is visible, mention the token but do not invent the URL.

PR links:

- If commit decorations or branch naming expose a PR number and `origin` points to GitHub, derive the PR URL from the remote.
- If a full PR URL appears in commit text or notes, preserve it.
- If a PR number exists but repo URL is ambiguous, mention `PR #123` style only.

## Suggested Formats

### Standup

```text
Yesterday:
- ...

Today:
- ...

Blockers:
- ...
```

### End-of-Day

```text
Done today:
- ...

In progress:
- ...

Next:
- ...
```

### Jira / Asana Comment

```text
Completed today:
- ...

Also:
- ...

Next:
- ...
```

## When To Ask A Follow-Up

Ask only if blocked on missing intent, for example:

- user wants Jira vs standup tone
- user wants a specific time window
- user wants personal vs team-facing wording

If not blocked, choose a sensible default and proceed.
