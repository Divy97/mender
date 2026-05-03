# RaftLabs Claude Code Plugin

The official RaftLabs toolkit for Claude Code. Install once per machine and get all RaftLabs skills in every project.

## Install

```bash
claude plugin add Raft-Labs/claude-plugins
```

All `raftkit-*` skills and both MCP servers (Asana + Playwright) are registered automatically.

**First-time Asana setup:** On first use of any Asana-dependent skill, Claude prompts you to authenticate via OAuth in your browser. One-time step.

## Update

```bash
claude plugin update raftlabs
```

Run after any merge to `main` to pull the latest skills.

## Skills

All skills are imported verbatim from the mentor project (`.agents/skills/raftkit-*`) and invoked as slash commands.

### Workflow
| Skill | Description |
|-------|-------------|
| `/raftkit-dev` | Task-driven development — reads task spec, plans, builds with TDD, commits incrementally, creates PR |
| `/raftkit-bug` | Universal bug resolution — investigate root cause, propose fix, optionally implement |
| `/raftkit-debug-issue` | Targeted debugging workflow for a specific reported issue |
| `/raftkit-release-note` | Generate release notes from git history and publish to Asana |
| `/raftkit-review-changes` | Review pending changes on the current branch |
| `/raftkit-refactor-safely` | Safe, incremental refactoring with verification |
| `/raftkit-explore-codebase` | Map an unfamiliar codebase before making changes |
| `/raftkit-verify-story` | Verify a completed story against its acceptance criteria |

### Domain
| Skill | Description |
|-------|-------------|
| `/raftkit-backend` | Lambda, Hono, Express, Zod patterns |
| `/raftkit-database` | Drizzle ORM, PostgreSQL migrations, indexes |
| `/raftkit-react` | React components, hooks, Next.js patterns |
| `/raftkit-mobile-dev` | Expo / React Native development workflow |
| `/raftkit-tdd` | Red-Green-Refactor cycle enforcement |
| `/raftkit-seo` | Metadata, OpenGraph, JSON-LD, Core Web Vitals |
| `/raftkit-code-quality` | Refactoring, function decomposition, complexity reduction |
| `/raftkit-asana` | Asana project management — invoke before any Asana MCP operation |

## Bundled MCP Servers

| MCP | Purpose |
|-----|---------|
| `@asana/mcp` | Asana task management — OAuth on first use |
| `@playwright/mcp` | Browser automation for UI verification |

## Roadmap

This is v1 — the consolidation step. Coming next:

- Pull additional skills from other `raftkit-*` source repos as they're identified
- Migrate Git hooks / commit conventions / branch protection from raftstack into this plugin
- Add the 4-stage Website Prompt System (`/build`, `/judge`, `/feedback`, `/polish`) once finalised
- Versioning convention TBD

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
