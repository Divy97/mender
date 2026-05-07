# Raftstack review + plan for the mender plugin

## TL;DR

The two raftstack branches are different products, not divergent versions — `main` is the npm CLI doing git plumbing, `raftkit_v1` is a Claude plugin built around a heavy 5-phase workflow. They don't merge cleanly.

Cleaner path: **mender becomes the single source of truth.** Port `main`'s git-scaffolding configs in as static templates (no rewriting, just lift the rendered files), cherry-pick ~15 useful skills from `raftkit_v1`, and skip the 5-phase workflow entirely. End state: one `claude plugin install` and one `/raftkit-init` instead of npm CLI + plugin.

`@raftlabs/raftstack` on npm stays untouched — we just stop using it ourselves.

---

## What's on each branch

### `raftstack/main` — npm CLI (`@raftlabs/raftstack` v1.10.5)

- TypeScript CLI. `npx @raftlabs/raftstack init` per project.
- Owns the *git plumbing*: Husky, commitlint, cz-git, lint-staged, Prettier, ESLint, branch validation, GitHub Actions, CODEOWNERS, PR templates, AI code review (CodeRabbit/Copilot).
- Side-ships 7 skills + 8 RaftStack commands into the target project's `.claude/`.
- **Asset to keep**: the rendered config files (`src/generators/*` outputs). Battle-tested.
- **Limitation**: distribution is npm; updates require rerunning the CLI in every project. Exactly the friction we want to remove.

### `raftstack/raftkit_v1` — Claude Code plugin

- Native plugin (`.claude-plugin/plugin.json` + `marketplace.json`). No npm.
- Big content library: 9 commands, 4 main agents + 23 review agents, 48 skills, MCP configs.
- Built around a 5-phase autonomous workflow: **Init → Blueprint → Plan → Build → Review → Compound**. Heavy, opinionated, phase-gated.
- **Asset to keep**: ~15 of the 48 skills (stack skills + meta-skills).
- **Asset to drop**: the 5-phase workflow itself — over-engineered shape we want to avoid. Skip everything tied to it (`blueprint`, `compound`, `constitutional-enforcement`, `cross-phase-validation`, the 23 review agents, `auto`, `status`).

The branches don't merge cleanly because they answer different questions. Each owns one job.

---

## Direction: mender absorbs both, single plugin

Three-step install becomes two:

1. `better-t-stack` scaffolds the repo (unchanged, external).
2. `claude plugin install raftlabs/mender` → user runs `/raftkit-init` inside their repo. That one command does both the git-plumbing (Step 2 from the call) *and* the Claude setup (Step 3 from the call).

Inside mender:

```
plugins/raftlabs/
├── commands/
│   ├── raftkit-init.md          # Git plumbing: husky/commitlint/CI/CODEOWNERS
│   ├── raftkit-setup.md         # CLAUDE.md + settings.json + superpowers, etc.
│   ├── raftkit-dev.md           # Promoted from skill → command
│   ├── raftkit-bug.md           # Promoted from skill → command
│   └── raftkit-release-note.md  # Promoted from skill → command
├── skills/                      # Auto-fire skills only (react, backend, db, asana, tdd…)
└── templates/                   # Static configs lifted from raftstack/main
    ├── husky/
    ├── commitlint.config.js
    ├── .github/workflows/*.yml
    └── ...
```

**Key call**: `raftkit-init` does **not** ask Claude to generate Husky/commitlint/CI from scratch. It bundles the rendered files as static templates (lifted directly from `raftstack/main`'s `src/generators/` output) and the command body just tells Claude "detect project type, copy these templates, merge package.json scripts." Determinism preserved, no token spend on solved problems. Detection logic (NX/Turbo/monorepo/React/Next) ships as a small `detect.sh` the command reads.

---

## In First phase

- We can convert `raftkit-dev`, `raftkit-bug`, `raftkit-release-note` from skills → commands. Write `/raftkit-setup` for CLAUDE.md / settings.json / dependency skills. 

---
