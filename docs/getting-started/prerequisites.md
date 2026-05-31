# Prerequisites

Three things are required before installing GoalBuddy.

## Node.js >= 18

GoalBuddy's CLI runs on Node.js 18 or later.

```bash
node --version
```

Expected output: `v18.0.0` or higher (e.g., `v22.3.0`). If the command is not found or returns a lower version, install or update Node.js from [nodejs.org](https://nodejs.org/).

## A supported AI coding agent

GoalBuddy installs into Codex, Claude Code, or both. You need at least one.

| Agent | GoalBuddy entry point |
|---|---|
| OpenAI Codex | `$goal-prep` |
| Anthropic Claude Code | `/goal-prep` |

> **Note:** Native Codex `/goal` is a separate OpenAI-gated feature. GoalBuddy prepares boards and handoff prompts for it; it does not enable or replace it.

## No other dependencies

GoalBuddy has no additional runtime dependencies. `npx goalbuddy` downloads and runs the package directly from npm. No global install is required.

## Verification

After installing GoalBuddy (see [quickstart](quickstart.md)), verify the Codex install:

```bash
npx goalbuddy doctor --target codex --goal-ready
```

To check Claude Code:

```bash
npx goalbuddy doctor --target claude
```
