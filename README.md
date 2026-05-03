# RaftLabs Claude Code Plugin

The RaftLabs toolkit for Claude Code. Install once per machine and get all RaftLabs skills in every project.

> **Status:** v1 lives at `Divy97/mender` while we consolidate. It will move to `Raft-Labs/claude-plugins` once the full skill set is migrated.

## Install

Inside Claude Code, run:

```
/plugin marketplace add Divy97/mender
/plugin install raftlabs@raftlabs-plugins
/reload-plugins
```

All 16 `raftkit-*` skills and both MCP servers (Asana + Playwright) are registered automatically. Verify with `/help` — you should see entries like `/raftlabs:raftkit-dev`.

**First-time Asana setup:** On first use of any Asana-dependent skill, Claude prompts you to authenticate via OAuth in your browser. One-time step.

## Update

After a new commit lands on `main`, pull it in with:

```
/plugin update raftlabs
/reload-plugins
```

## Invoking skills

Skills are namespaced under the plugin. The full form is:

```
/raftlabs:<skill-name>
```

For example: `/raftlabs:raftkit-dev`, `/raftlabs:raftkit-bug`, `/raftlabs:raftkit-asana`.

## Skills

All skills are imported verbatim from the mentor project (`.agents/skills/raftkit-*`).

### Workflow
| Skill | Description |
|-------|-------------|
| `/raftlabs:raftkit-dev` | Task-driven development — reads task spec, plans, builds with TDD, commits incrementally, creates PR |
| `/raftlabs:raftkit-bug` | Universal bug resolution — investigate root cause, propose fix, optionally implement |
| `/raftlabs:raftkit-debug-issue` | Targeted debugging workflow for a specific reported issue |
| `/raftlabs:raftkit-release-note` | Generate release notes from git history and publish to Asana |
| `/raftlabs:raftkit-review-changes` | Review pending changes on the current branch |
| `/raftlabs:raftkit-refactor-safely` | Safe, incremental refactoring with verification |
| `/raftlabs:raftkit-explore-codebase` | Map an unfamiliar codebase before making changes |
| `/raftlabs:raftkit-verify-story` | Verify a completed story against its acceptance criteria |

### Domain
| Skill | Description |
|-------|-------------|
| `/raftlabs:raftkit-backend` | Lambda, Hono, Express, Zod patterns |
| `/raftlabs:raftkit-database` | Drizzle ORM, PostgreSQL migrations, indexes |
| `/raftlabs:raftkit-react` | React components, hooks, Next.js patterns |
| `/raftlabs:raftkit-mobile-dev` | Expo / React Native development workflow |
| `/raftlabs:raftkit-tdd` | Red-Green-Refactor cycle enforcement |
| `/raftlabs:raftkit-seo` | Metadata, OpenGraph, JSON-LD, Core Web Vitals |
| `/raftlabs:raftkit-code-quality` | Refactoring, function decomposition, complexity reduction |
| `/raftlabs:raftkit-asana` | Asana project management — invoke before any Asana MCP operation |

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
5. Team runs `/plugin update raftlabs` then `/reload-plugins` inside Claude Code

## Contributing

- Anyone at RaftLabs can raise a PR
- Divy reviews and approves all merges to `main`
- Use `# TODO: replace with your value` for any project-specific placeholders — never hardcode project GIDs, URLs, or tokens
- Test locally before raising a PR (see [Local Testing](#local-testing))

## Local Testing

To test plugin changes from a local checkout before pushing:

1. Commit your changes locally (the marketplace loader needs them on a branch):

   ```bash
   git add . && git commit -m "wip"
   ```

2. Inside Claude Code, register the local repo as a marketplace:

   ```
   /plugin marketplace add file:///Users/you/path/to/mender
   /plugin install raftlabs@raftlabs-plugins
   /reload-plugins
   ```

3. After further changes, commit and reload:

   ```
   /plugin update raftlabs
   /reload-plugins
   ```
