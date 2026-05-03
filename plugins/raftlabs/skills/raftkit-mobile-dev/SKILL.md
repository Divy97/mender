---
name: raftkit-mobile-dev
description: >-
  Mobile development workflow for the React Native app. Combines /raftkit-dev rigor
  (status audit, plan-mode entry, TDD, negative-scenario analysis, verification
  suite) with /raftkit-bug flexibility (multi-entry input parsing — task spec, Asana,
  URL, error trace, free text, interactive — and root-cause investigation).
  Wraps every code change with the device See → Code → Verify screenshot loop
  via the agent-device skill. NEVER creates a branch — assumes the user is
  already inside a git worktree on a feature/fix/bugfix/hotfix/chore branch.
  NEVER pushes or opens a PR — atomic commits per confirmed fix only; the user
  opens the PR when ready. Use this skill for ANY mobile work — features, bug
  fixes, styling, layout, navigation, refactoring visible UI — even when the
  user just says "fix this" or "change that" and mobile is the context.
  Invoke: /raftkit-mobile-dev @path/to/task.md, /raftkit-mobile-dev <asana-link>,
  /raftkit-mobile-dev <description>, /raftkit-mobile-dev (interactive).
---

# Mobile Dev Workflow

Combines `/raftkit-dev`'s discipline with `/raftkit-bug`'s entry flexibility, scoped to the React Native app at `apps/mobile/` and the shared `@mentor/ui-mobile` package. Keeps the See → Code → Verify device loop as the unique mobile mechanism (it replaces `/raftkit-dev`'s Playwright phase, which can't drive React Native).

## What stays the same as the old mobile-dev

- The session-style ergonomics: many bugs in one sitting, atomic commit per confirmed fix, agent-device session opened once per conversation.
- The trigger phrases: "okay it is fixed", "that's fixed", "looks good", "next bug" → commit the prior fix and start the next investigation.
- The dev-overlay handling rules from `agent-device` (dismiss if non-blocking, fix and report otherwise).

## What's new

- Plan-mode entry on every invocation (Phases 1-2 are read-only).
- Multi-entry input parsing — task spec, Asana, URL, error trace, free text, interactive — modeled on `/raftkit-bug`.
- Status audit + research + written plan before code (Phase 1, Phase 2).
- Mandatory negative-scenario analysis (Phase 4b) with mobile-flavoured categories.
- Verification suite (lint + types + tests) before any "done" claim (Phase 6).
- Branch verification (no branch creation — see Hard Gates).
- TDD on testable units (hooks, validators, utilities, navigation guards, reducers); device verification for visual UI.

## Assumptions

The app is already running. The user has done:

1. `pnpm -F mobile prebuild`
2. `pnpm dev:mobile` (Metro is up)
3. `pnpm -F mobile android` or `pnpm -F mobile ios`

Do NOT prebuild, restart Metro, or reinstall the app. The user has already chosen the device and branch — you're a guest.

## agent-device session setup

On the first device interaction of the conversation only, run:

```
agent-device open com.mybeautymentors.app --platform <android|ios>
```

This binds agent-device to the already-running app process — it does not restart anything. After that, every screenshot/snapshot/press/wait/scroll/logs command works without re-opening.

---

## Input parsing

Parse the `/raftkit-mobile-dev` invocation in this order. The first match wins.

| Entry type      | Detection                                                                                 | Example                                                                    |
| --------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **task-spec**   | First arg starts with `@` and ends with `.md`                                             | `/raftkit-mobile-dev @docs/project-management/05-courses/course-screen.md` |
| **asana**       | URL contains `app.asana.com`, OR raw 10+ digit numeric string                             | `/raftkit-mobile-dev https://app.asana.com/0/0/1234567890/f`               |
| **url**         | Starts with `http://` / `https://` and is NOT an Asana URL                                | `/raftkit-mobile-dev https://staging.mentor.app/courses/123`               |
| **error-trace** | Contains stack-trace markers (`Error:`, `at `, `.ts:`, `.tsx:`, file path + line numbers) | `/raftkit-mobile-dev TypeError: ... at LoginScreen (login.tsx:42)`         |
| **description** | Any other non-empty text                                                                  | `/raftkit-mobile-dev the auth hero clips on the right edge`                |
| **interactive** | Empty invocation                                                                          | `/raftkit-mobile-dev`                                                      |

### Rules

- For `task-spec`, set `taskFile = path` and run the `/raftkit-dev`-style flow (full status audit, written plan, TDD, verification suite). Asana link and PROGRESS.md update apply.
- For `asana`, MUST invoke the `/raftkit-asana` skill first before any Asana MCP call (mandatory per project memory). Then fetch the task + comments and treat QA observations as bug context.
- For `url`, Playwright is unavailable for native — instead, ask the user to navigate to the matching screen on the device, then `screenshot` + `snapshot` to capture state.
- For `error-trace`, parse referenced files and read the source at the cited line numbers before forming a hypothesis.
- For `description` and `interactive`, prompt for missing detail (expected vs actual, reproduction).
- If the entry looks like an Asana URL but the GID can't be extracted, ask the user to re-paste.

Always also ask once: "Is there an Asana task for this? (paste link, or 'no')" if `asanaGid` is not yet set. Store it for Phase 7.

---

## Branch verification

Before doing anything else, confirm we're on a working branch.

1. Run `git branch --show-current`.
2. **If on `development`, `main`, `master`, `staging`, or `production`** — STOP. Tell the user:
   "I won't run on `<branch>`. Switch into your mobile worktree (or another working branch) and re-invoke `/raftkit-mobile-dev`. This skill never creates branches."
3. **If on a working branch** (`feature/*`, `fix/*`, `bugfix/*`, `hotfix/*`, `chore/*`, `refactor/*`, `test/*`, `ci/*`, `docs/*`, `release/*`) — record the branch and proceed.

<HARD-GATE>
NEVER create a branch in this skill. The user is in a worktree by design — every commit lives on the current branch.
</HARD-GATE>

---

## Auto plan mode

Always EnterPlanMode at the start of every invocation. Plan mode covers Phase 1 (Context & Status Audit) and Phase 2 (Research & Plan). After the user approves the plan, ExitPlanMode and proceed to Phase 4+.

If `EnterPlanMode` is unavailable, instruct the user: "Please type `/plan` to enter plan mode, then re-invoke `/raftkit-mobile-dev`."

<HARD-GATE>
NEVER write implementation code while in plan mode. Phases 1-2 are read-only.
</HARD-GATE>

---

## Phase 1 — Context & Status Audit

**Goal:** Build a complete picture of the work, regardless of how it was reported, and capture the device's starting visual state.

### Branch by entry type

#### 1a. task-spec

Mirror `/raftkit-dev` Phase 1:

1. Read the task file. Extract title, description, affected apps/packages, requirements, acceptance criteria, dependencies, technical notes.
2. Check upstream dependencies via Explore agents — warn if upstream work is missing.
3. For each acceptance criterion, search the codebase for evidence and classify Done / Partial / Not Started.
4. Present a status matrix.

#### 1b. asana

Mirror `/raftkit-bug` 1a:

1. Invoke `/raftkit-asana` (mandatory).
2. `asana_get_task` with full opt_fields, then `asana_get_stories_for_task` filtered to comments.
3. Present issue summary: title, permalink, assignee, due date, section, tags, description, chronological QA comments.

#### 1c. url

Playwright cannot drive React Native. Adapt the URL to a device interaction:

1. Identify the matching screen (search by route name in `apps/mobile/app/`).
2. Ask the user to navigate the device to that screen if it isn't already on it.
3. `screenshot` + `snapshot` to capture state.
4. Note any visible errors or unexpected UI.

#### 1d. error-trace

Mirror `/raftkit-bug` 1c:

1. Parse error type, message, file paths, line numbers.
2. Read source files at the referenced locations (use Read for each `file:line`).
3. Present error analysis with surrounding code context.

#### 1e. description / interactive

Mirror `/raftkit-bug` 1d:

1. If interactive, prompt: "Describe the bug or change. Include expected vs actual and reproduction steps."
2. Acknowledge the description. Ask whether there's an Asana task.

### Visual anchor (always, when a device is connected)

After the entry-specific work, take a starting `screenshot` of the device (and `snapshot` if the task touches specific UI elements) so we have ground truth before any code changes. Briefly describe what's on screen to confirm shared context. This is the "See" half of See → Code → Verify, hoisted into Phase 1.

Skip the screenshot only when:

- No device is connected (`devices` returns nothing) — note this and continue.
- The task is purely non-visual (utility function, type-only change, config) — note this and continue.
- The user explicitly says "skip the device" or similar.

### Asana link prompt

If `asanaGid` is not yet set, ask once: "Is there an Asana task for this? (paste URL, GID, or 'skip')". Extract the GID using the URL parsing rules from `/raftkit-dev` Phase 1 step 6. Store as `asanaGid` and `asanaUrl` for Phase 7. In Asana entry mode, the GID is already set — skip this prompt.

<HARD-GATE>
NEVER skip the status audit for task-spec entries. Read the full task file and assess every acceptance criterion before planning.
</HARD-GATE>

---

## Phase 2 — Research & Plan

**Goal:** Verify every technical decision against current docs, design the approach, write a plan, and get user approval.

### Research protocol

Apply `/raftkit-dev`'s Research Protocol — Context7 first for any library/API/pattern, specialized MCPs for matching domains, WebSearch as last resort. Mobile-leaning hierarchy:

| Domain                         | Primary tool                                   | Fallback             |
| ------------------------------ | ---------------------------------------------- | -------------------- |
| Expo / EAS / config plugins    | `mcp__expo-mcp`, `expo-deployment` skill       | Context7 → WebSearch |
| HeroUI Native                  | `heroui-native` skill                          | Context7 → WebSearch |
| Expo Router / native UI        | `building-native-ui` skill                     | Context7 → WebSearch |
| React Native networking        | `native-data-fetching` skill                   | Context7 → WebSearch |
| Better Auth (mobile auth flow) | `mcp__better-auth` tools                       | Context7 → WebSearch |
| Tailwind via Uniwind           | `expo-tailwind-setup` skill                    | Context7             |
| Any npm package                | Context7 (`resolve-library-id` → `query-docs`) | WebSearch → WebFetch |

Present every finding to the user using the Research Finding or Decision Brief format from `/raftkit-dev`. Wait for approval on each decision before continuing.

### Brainstorm and plan

1. Invoke `superpowers:brainstorming` to explore the design space and surface alternatives.
2. Run a Design Interview only when the work spans more than one file or one screen — for a 3-line visual fix, a 3-line plan is enough. For multi-step features and bug investigations with non-obvious root causes, resolve every decision branch before writing the plan.
3. Invoke `superpowers:writing-plans`. Save to `docs/plans/YYYY-MM-DD-<slug>-mobile-plan.md` (the `-mobile-` infix avoids collisions with concurrent `/raftkit-dev` plans).
4. Plan content: ordered steps, files to create/modify, test strategy, libraries verified, research log. For trivial fixes, three lines: "what + where + verify on device" is enough.
5. Get explicit user approval. Then ExitPlanMode.

<HARD-GATE>
NEVER proceed to implementation without an approved plan. The user must confirm before any code is written.
</HARD-GATE>

---

## Phase 4 — TDD Implementation

**Goal:** Implement the work test-first for testable units; verify visually for pure-presentational UI.

### TDD targets (test-first is mandatory)

- Hooks (custom hooks in `apps/mobile/lib/`, `apps/mobile/hooks/`)
- Validators (`@mentor/validators`)
- Utilities (`@mentor/utils`, mobile-local helpers)
- Navigation guards / route handlers
- State reducers, derived state computations
- Business logic separated from rendering

### Visual-only changes (verify on device)

Pure-presentational tweaks — colors, spacing, typography, layout, copy — are verified in Phase 6b on the device. Tests at the render level for these are usually noise; rely on the screenshot loop.

### Skill auto-detection

Scan the task content for signals and announce which mobile skills will be invoked.

| Signal                                            | Skills to invoke                                             |
| ------------------------------------------------- | ------------------------------------------------------------ |
| Screen, route, navigation, Expo Router            | `building-native-ui`                                         |
| HeroUI Native component                           | `heroui-native`                                              |
| Tailwind / Uniwind / NativeWind                   | `expo-tailwind-setup`                                        |
| Network / API call / fetch / TanStack Query       | `native-data-fetching`                                       |
| iOS / Android build / EAS / signing / entitlement | `expo-deployment`, `expo-cicd-workflows`                     |
| Native module / config plugin / Swift / Kotlin    | `expo-module`, `expo-ui-swift-ui`, `expo-ui-jetpack-compose` |
| DOM in WebView                                    | `use-dom`                                                    |
| Mobile auth / SecureStore / sessions              | `better-auth-best-practices`                                 |
| API / oRPC / Hono changes touched by mobile       | `backend`, `hono`                                            |

Always active: `tdd`, `code-quality`.

### Implementation loop

For each acceptance criterion (or each finding from `/raftkit-bug`-style investigation):

0. **Research check** — pause if the criterion involves a first-time library API or unplanned dependency.
1. Write failing tests (Red) — for testable units.
2. Write minimal implementation to pass (Green).
3. Refactor while green.
4. After Edit/Write, immediately go to Phase 6b for that change before moving on (visual confirmation must happen now, not at the end).
5. Mark the criterion's status; update the running list.

### Mid-implementation research triggers

Same as `/raftkit-dev`: first-time API call, library error, unplanned dependency, training-data pattern → pause, research, present findings, resume.

---

## Phase 4b — Negative Scenario Analysis (mobile-flavoured)

**Goal:** Proactively identify failure modes against the code just implemented. Every gap becomes another TDD cycle.

<HARD-GATE>
NEVER skip Phase 4b. Even if every acceptance criterion passes, the code may have unhandled failure modes. Mobile UIs in particular fail badly on offline, denied permissions, and platform deltas.
</HARD-GATE>

### When it runs

After all acceptance criteria are `done` (or `partial` with user approval), before the final commit batch.

### The 8 categories

1. **Error states** — API failures (oRPC errors), DB errors surfaced via toast/alert, ErrorBoundary present where useful, no silent swallowing.
2. **Empty / null / undefined** — empty list states, optional fields guarded with `?.` / `?? default`, loading skeletons, `noUncheckedIndexedAccess` array guards.
3. **Boundary conditions** — long text wraps within safe area, max-length validators match server constraints, emoji + RTL safe.
4. **Auth / permission** — `protectedProcedure` on the API, SecureStore session restored on cold start, deep-link handling when not signed in, role checks at the API.
5. **Concurrency** — double-tap submit guarded with `isPending` / disabled, back-navigation during async cleaned up, TanStack Query cache invalidation after mutations.
6. **Network failures** — offline behaviour, retries, timeouts surfaced, slow-network skeletons.
7. **Platform & device differences** — iOS vs Android safe-area / paddings / status-bar, light/dark theme, small phone vs tablet, haptics availability, keyboard avoidance.
8. **Native permissions** — camera / photos / location / notifications: graceful denial, re-prompt UX, settings-deep-link fallback.

### Output

```
### Negative Scenario Analysis

| # | Category | Relevant? | Handled? | Finding |
|---|----------|-----------|----------|---------|
| 1 | Error states | Yes | No | No toast on signup error |
| 2 | Empty data | Yes | Yes | Empty state component renders |
| 3 | Boundary | N/A | — | No text input in this flow |
| 4 | Auth | Yes | Yes | protectedProcedure in place |
| 5 | Concurrency | Yes | No | Submit button not disabled during mutation |
| 6 | Network | Yes | Partial | Error caught but no user-visible message |
| 7 | Platform | Yes | No | iOS uses 44pt safe-area inset, Android uses 0 |
| 8 | Permissions | N/A | — | No native permission required here |
```

Every "No" or "Partial" → another Red-Green TDD cycle plus a device verification pass. Re-assess the table after fixing.

---

## Phase 5 — Atomic Commits

**Goal:** Commit at each user confirmation. One logical change per commit.

### Trigger

The user says "okay it is fixed", "that's fixed", "looks good", "next bug", or similar confirmation → commit the prior fix immediately, then move to the next investigation. This pattern is mobile-dev's signature — keep it.

### Format

```
<emoji> <type>(<scope>): <subject>
```

- Emoji: Unicode character (✨ 🐛 ✅ ♻️ 📝 🔧 ⚡️ 📦 🎡 💄 ⏪), not `:shortcode:`.
- Type: `feat`, `fix`, `test`, `refactor`, `docs`, `chore`, `perf`, `build`, `ci`, `style`, `revert`.
- Scope: most commonly `mobile` or `ui-mobile` for this skill. Cross-package edits use the matching workspace scope (`api-pkg`, `auth`, `db`, `validators`, etc.).
- Subject: lowercase, imperative, max 100-char header.
- No `Co-Authored-By` trailer. No extra footers unless breaking change.

### Procedure

1. Stage only the files modified for this fix — never `git add -A` or `git add .`.
2. Generate the message. If you're unsure of the subject, ask the user once before committing.
3. Commit via HEREDOC:
   ```bash
   git commit -m "$(cat <<'EOF'
   🐛 fix(mobile): blend auth hero into cream background
   EOF
   )"
   ```
4. Acknowledge briefly (one line) and wait for the next task.

### Examples

```
🐛 fix(mobile): dismiss keyboard on signup submit
✨ feat(mobile): add empty state to bookmarks screen
♻️ refactor(ui-mobile): extract avatar component
✅ test(mobile): cover signup validation edge cases
🔧 chore(mobile): disable user script sandboxing for cocoapods
```

---

## Phase 6 — Verification

**Goal:** Run the static and unit-test suite before claiming done. Visual verification happens in Phase 6b.

### Suite

```bash
pnpm -F mobile lint
pnpm -F mobile check-types        # or root pnpm check-types if cross-package edits
pnpm -F mobile test               # if the package has tests; otherwise note "no tests"
```

If shared `@mentor/*` packages were edited, also `pnpm -F <pkg> test` and `pnpm build` for those packages. Skip `pnpm build` for mobile-only edits — Metro is the source of truth in dev. Skip `pnpm -F mobile android/ios` — the app is already running.

### Failure loop

If any check fails: fix → re-commit (Phase 5 format) → re-run the suite. Repeat until green.

<HARD-GATE>
NEVER claim done without lint + types + relevant tests green. The pre-push hook will reject anyway.
</HARD-GATE>

---

## Phase 6b — Device Verification (See → Code → Verify)

**Goal:** Confirm visual changes actually rendered the way you intended on a real device. This is mobile-dev's signature mechanism. It runs after every code change, not just at the end.

### Loop

1. Wait 2-3 seconds for Metro Fast Refresh to apply the change.
2. `screenshot` the device.
3. If the affected screen isn't currently displayed: `snapshot -i` to see interactive elements, then `press @ref` to navigate. Wait for the screen to settle, then screenshot.
4. Compare before vs after — did the change take effect as expected?
5. If wrong: fix the code → wait → screenshot. Cap at 3 verification rounds. After 3, ask the user what they see or whether there's an error overlay.
6. Dismiss / report any RN dev overlays per agent-device rules. Mention what you saw.
7. For each UI-visible acceptance criterion, attach the screenshot path as `uiEvidence`.

### Skip device verification when

- No device is connected (`devices` returns empty).
- The change is purely non-visual (types, hooks with no UI surface, utility functions, config files).
- The user explicitly says "don't screenshot" or similar.

When skipping, tell the user you couldn't verify visually so they know to spot-check.

<HARD-GATE>
NEVER run Playwright in this skill. Playwright cannot drive React Native — use agent-device.
</HARD-GATE>

---

## Phase 7 — Post-fix actions (no PR phase)

This skill never pushes or opens a PR. The user is on a long-running worktree branch and opens the PR via `/commit-push-pr` or `gh pr create` when they decide the branch is ready.

### PROGRESS.md (task-spec entries only)

For task-spec mode, after the last commit of the session: open `docs/project-management/PROGRESS.md`, mark the task `- [x]`, and append ` — branch <branch-name>` (no PR number yet). Commit:

```
📝 docs(docs): mark <task-slug> as complete on mobile branch
```

For all other entry types (asana, url, error-trace, description, interactive), skip PROGRESS.md — those are bug-style flows and don't own progress tracking.

### Asana update (if `asanaGid` is set)

Post a non-technical progress comment via `asana_create_task_story`. Tone follows `/raftkit-bug` Phase 7: written for QA / PM, no file paths, no function names, no code, no developer jargon. Describe what changed in user-facing terms.

```
Progress update from mobile dev workflow.

Branch: <branch-name>
Status: <fully-fixed-pending-pr | partially-fixed>

What was addressed:
- <user-facing description of fix 1>
- <user-facing description of fix 2>

<If negative-scenario findings were addressed:>
Related improvements: We also fixed edge cases discovered during investigation:
- <user-facing description>

<If anything is still outstanding:>
Still outstanding:
- <user-facing description>

PR will be opened once the branch is ready.
```

Do NOT close the task — only the eventual PR-merge or an explicit `/raftkit-asana` close handles that. Track within-session idempotency so the same comment isn't double-posted.

### Session cleanup

This skill does not write a `.dev-session.json` (we don't own the persistence checkpoint — `/raftkit-dev` does, and the bug/free-text flows don't need it). Hold session state in conversation memory only. If you also invoked `/raftkit-dev` in the same conversation and it created a session file, leave it alone.

<HARD-GATE>
NEVER push or open a PR from this skill. The user controls when the branch is ready.
</HARD-GATE>

<HARD-GATE>
NEVER auto-close an Asana task from this skill. Comment only.
</HARD-GATE>

---

## Hard Gates Summary

```
1. NEVER create a branch in this skill — work on the current worktree branch.
2. NEVER write implementation code while in plan mode.
3. NEVER skip the device See → Code → Verify loop for visual changes.
4. NEVER run Playwright in this skill — Playwright cannot drive React Native.
5. NEVER skip Negative Scenario Analysis (Phase 4b).
6. NEVER write implementation code before tests for testable units (TDD on
   hooks / utilities / validators / navigation guards / reducers; visual-only
   components verified on device).
7. NEVER commit without commitlint format — Unicode emoji + type + valid scope
   + lowercase subject + no Co-Authored-By trailer.
8. NEVER auto-close an Asana task from this skill — comment only.
9. NEVER push or open a PR from this skill — the user opens the PR when ready.
10. NEVER use a library API, framework pattern, or package configuration based
    solely on training data. Always verify via the Research Protocol and
    present findings to the user before implementing.
11. NEVER skip the status audit for task-spec entries.
12. NEVER claim done without passing lint + check-types + relevant tests.
```

---

## Related skills

- **agent-device** — handles all device commands (screenshot, snapshot, press, fill, wait, scroll, logs). Auto-loaded.
- **/raftkit-asana** — mandatory before any Asana MCP call.
- **/raftkit-tdd** and **/raftkit-code-quality** — always active during Phase 4.
- **superpowers:brainstorming** and **superpowers:writing-plans** — used in Phase 2.
- **superpowers:dispatching-parallel-agents** — when investigation spans multiple areas.
- **/raftkit-dev** and **/raftkit-bug** — sibling workflows. `/raftkit-dev` is preferred when the work is API-heavy or fully task-spec-driven; `/raftkit-bug` is preferred for non-mobile bug investigations.
- **dogfood** — for systematic exploration / QA sessions, not feature development.
- **building-native-ui**, **heroui-native**, **native-data-fetching**, **expo-tailwind-setup**, **expo-deployment**, **expo-cicd-workflows**, **expo-module**, **upgrading-expo**, **use-dom** — auto-detected from task signals in Phase 4.

---

## Common mistakes to avoid

| Mistake                                 | Why it's wrong                                       | What to do instead                                        |
| --------------------------------------- | ---------------------------------------------------- | --------------------------------------------------------- |
| Creating a feature branch               | User is in a worktree; this skill assumes the branch | Skip branch creation; verify current branch only          |
| Pushing or opening a PR                 | Out of scope for this skill                          | Stop after the last commit; user opens PR when ready      |
| Running Playwright                      | Cannot drive React Native                            | Use agent-device                                          |
| Writing code in plan mode               | Plan mode is research-only                           | Exit plan mode after Phase 2 approval                     |
| Skipping the starting screenshot        | Loses ground truth                                   | Screenshot in Phase 1 unless non-visual / no device       |
| Skipping Phase 4b                       | Mobile fails on offline / permissions / platform     | Always run the 8-category table                           |
| Treating every UI tweak as TDD-required | Render tests for visuals are noise                   | Visual-only changes verify on device; logic units use TDD |
| Verifying only at the end               | Three changes deep, you've lost the trail            | Verify on device after each Edit/Write                    |
| `git add -A`                            | Stages unrelated files, possibly secrets             | Stage specific files by name                              |
| `:shortcode:` emoji in commits          | commitlint expects Unicode                           | Use ✨ not `:sparkles:`                                   |
| Closing the Asana task automatically    | Misrepresents completion before merge                | Comment only — closure happens on merge                   |
| Posting duplicate Asana comments        | Creates noise                                        | Track within-session that the comment was posted          |
| Using a library API from memory         | Training data may be stale                           | Run the Research Protocol first; present findings         |
| Bypassing hooks with `--no-verify`      | Hides real failures                                  | Fix the underlying issue                                  |
