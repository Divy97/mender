# Mender — RaftLabs Claude Code Plugin Repo

This repo is the plugin container. It is not a product — it is the toolkit that powers Claude Code across all RaftLabs projects.

## Structure

```
mender/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace listing — bump version here on every merge
└── plugins/
    └── raftlabs/
        ├── .claude-plugin/
        │   └── plugin.json       # Plugin metadata + MCP servers — bump version here too
        └── skills/
            └── <skill-name>/
                └── SKILL.md      # One file per skill
```

## Adding a Skill

1. Create `plugins/raftlabs/skills/<skill-name>/SKILL.md`
2. Required frontmatter:
   ```yaml
   ---
   name: skill-name
   description: >-
     One-line description. Explain when to invoke it, not what it does.
   ---
   ```
3. Bump `version` in both `plugin.json` and `marketplace.json` (semver: patch for content fixes, minor for new skills, major for breaking changes)
4. Raise PR

## Skill Writing Rules

- **Placeholders over hardcodes** — use `# TODO: replace with your Asana project GID` for any project-specific value. Never put real GIDs, URLs, or tokens in this repo.
- **MCP guards** — any phase that calls an MCP tool must open with a graceful degradation block (see `/dev` or `/bug` for examples).
- **No implementation comments** — skill content should explain *what to do and why*, not narrate the code.
- **One skill, one purpose** — if a skill needs to branch heavily based on context, consider splitting it.

## MCP Servers

The plugin auto-registers two MCP servers on install:
- `asana` — `@asana/mcp` via npx, OAuth auth
- `playwright` — `@playwright/mcp` via npx, no auth needed

To add another MCP server: add it to `plugins/raftlabs/.claude-plugin/plugin.json` under `mcpServers`.

## Versioning

Both files must stay in sync:
- `plugins/raftlabs/.claude-plugin/plugin.json` → `"version"`
- `.claude-plugin/marketplace.json` → `"version"`

## Merge Process

Anyone at RaftLabs can raise a PR. Divy reviews and merges to `main`. After merge, team runs `claude plugin update raftlabs`.
