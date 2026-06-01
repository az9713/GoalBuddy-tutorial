# GoalBuddy Plugin (Codex + Claude Code)

GoalBuddy packages the canonical `goal-prep` skill as a plugin so teams can install the reusable workflow in **Codex** and **Claude Code**, while keeping the npm CLI for local setup, doctor checks, and the built-in local board surface.

Version 0.3.8 keeps the Goalmaxxed core and adds board-hub guardrails so agents reuse the shared local board hub instead of stopping it after an unregistered board URL returns 404.

## What It Contains

- `.codex-plugin/plugin.json`: Codex plugin manifest and Codex UI copy.
- `.claude-plugin/plugin.json`: Claude Code plugin manifest.
- `skills/goalbuddy/`: the installable GoalBuddy skill payload (shared by both platforms).
- `agents/`: Claude Code subagent definitions (`goal-scout.md`, `goal-judge.md`, `goal-worker.md`).
- `skills/goalbuddy/SKILL.md`: canonical `$goal-prep` / `/goal-prep` entry point.
- `assets/goalbuddy-icon.svg`: lightweight plugin icon.

## Local Testing

From the repo root:

```bash
npm run check
npx goalbuddy doctor
npx goalbuddy check-update
```

## Install Both Targets

```bash
npx goalbuddy
```

This installs and enables the native Codex plugin in `~/.codex/`, then installs the GoalBuddy skill and Scout/Judge/Worker subagents into `~/.claude/`. The skill surfaces `/goal-prep` in Claude Code.

## Install One Target

```bash
npx goalbuddy --target codex
npx goalbuddy --target claude
```

This installs the GoalBuddy skill and the three Scout/Judge/Worker subagents into `~/.claude/`. Restart Claude Code, then run:

```text
/goal-prep
```

Or install the npm package globally:

```bash
npm i -g goalbuddy
goalbuddy                  # installs for Codex and Claude Code
goalbuddy --target codex   # installs for Codex only
goalbuddy --target claude  # installs for Claude Code only
```

For local CLI testing before npm publish:

```bash
node internal/cli/goal-maker.mjs
node internal/cli/goal-maker.mjs doctor
node internal/cli/goal-maker.mjs board docs/goals/<slug> --once --json
```

## Release Notes

The plugin is prepared for the `tolibear/goalbuddy` repo and `goalbuddy` npm package. Keep `.codex-plugin/plugin.json` and `.claude-plugin/plugin.json` aligned with `package.json` before publishing a new package release.
