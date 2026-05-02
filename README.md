# RaftLabs Claude Code Plugin

The official RaftLabs toolkit for Claude Code. Install once per machine and get all company skills and slash commands in every project.

## Install

```bash
claude plugin add Raft-Labs/claude-plugins
```

That's it. All 17 skills and both MCP servers (Asana + Playwright) are registered automatically.

**First-time Asana setup:** On first use of any Asana-dependent skill, Claude will prompt you to authenticate via OAuth in your browser. One-time step.

## Update

```bash
claude plugin update raftlabs
```

Run this after any merge to `main` to pull the latest skills.

## Skills

### Workflow
| Skill | Description |
|-------|-------------|
| `/dev` | Full task-driven development cycle — reads task spec, plans, builds with TDD, commits incrementally, creates PR |
| `/bug` | Universal bug resolution — investigate root cause, analyze negative scenarios, propose fix, optionally implement |
| `/release-note` | Generate release notes from git history and publish to Asana |

### Domain
| Skill | Description |
|-------|-------------|
| `/backend` | Lambda functions, Hono handlers, Express routes, Zod validation |
| `/database` | Drizzle ORM schemas, PostgreSQL migrations, indexes, relations |
| `/react` | React components, hooks, Next.js pages, Server/Client Components |
| `/tdd` | Enforces Red-Green-Refactor cycle, 80% minimum coverage |
| `/seo` | Metadata, OpenGraph, JSON-LD, Core Web Vitals, Next.js Metadata API |
| `/code-quality` | Refactoring, function decomposition, complexity reduction |
| `/asana` | Asana project management — must invoke before any Asana MCP operations |

### Utility
| Skill | Description |
|-------|-------------|
| `/dev-worklog` | Grounded developer status updates from local repo activity |
| `/qa-scenario-matrix` | Generate comprehensive QA scenarios for any feature |
| `/grill-me` | Relentless questioning about a plan until reaching shared understanding |

### 4-Stage Website System
Run these in sequence for any frontend/website work:

| Skill | Stage | Description |
|-------|-------|-------------|
| `/build` | 1 | Generate complete initial implementation from a brief |
| `/judge` | 2 | Structured critique — design, UX, copy, accessibility, code quality |
| `/feedback` | 3 | Incorporate stakeholder feedback into a concrete plan and execute |
| `/polish` | 4 | Micro-interactions, animations, accessibility, production readiness |

## Bundled MCP Servers

| MCP | Purpose |
|-----|---------|
| `@asana/mcp` | Asana task management — OAuth auth on first use |
| `@playwright/mcp` | Browser automation for UI verification |

## Adding a New Skill

1. Create `plugins/raftlabs/skills/<skill-name>/SKILL.md`
2. Add the required frontmatter:

```markdown
---
name: skill-name
description: >-
  One-line description shown in /help. Explain when to invoke it.
---

# Skill Title

...skill content...
```

3. Bump `version` in both `plugins/raftlabs/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
4. Raise a PR — Divy reviews and merges
5. Team runs `claude plugin update raftlabs`

## Contributing

- Anyone at RaftLabs can raise a PR
- Divy reviews and approves all merges to `main`
- Use `# TODO: replace with your value` for any project-specific placeholders — never hardcode project GIDs, URLs, or tokens
- Test locally before raising a PR (see [Local Testing](#local-testing))

## Local Testing

To test plugin changes before pushing to GitHub:

```bash
# 1. Init git if not already done
cd /path/to/mender
git init && git add . && git commit -m "changes"

# 2. Add to ~/.claude/settings.json under extraKnownMarketplaces:
# "raftlabs-local": {
#   "source": { "source": "git", "url": "file:///path/to/mender" }
# }

# 3. Install
claude plugin add raftlabs --from raftlabs-local

# 4. After further changes, reinstall
claude plugin remove raftlabs && claude plugin add raftlabs --from raftlabs-local
```
