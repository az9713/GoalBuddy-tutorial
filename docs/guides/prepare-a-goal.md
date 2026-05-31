# Prepare a goal

Run `goal-prep` to convert a raw intent into a structured GoalBuddy board before starting a `/goal` run. Use it for any task that is broad, multi-hour, vague, high-risk, stale, or likely to need Scout/Judge/Worker delegation.

## Prerequisites

| Requirement | How to satisfy |
|---|---|
| GoalBuddy installed | `npx goalbuddy` then restart Codex or Claude Code |
| Active Codex or Claude Code session | Open the project in Codex or Claude Code |
| Node.js available in the shell | `node --version` must succeed |

## Steps

### 1. Invoke goal-prep

In **Codex**:

```text
$goal-prep
```

In **Claude Code**:

```text
/goal-prep
```

`goal-prep` first checks whether a newer GoalBuddy version is available and prints an update notice without blocking if one is found.

### 2. Answer the diagnostic intake (vague goals only)

For vague, strategic, or improvement-oriented goals, `goal-prep` runs a guided intake one question at a time. The private intake document is never shown; you see only the current question.

The diagnostic ladder covers five topics in order:

| Step | Topic | What gets resolved |
|---|---|---|
| 1 | Goal surface | Whether to open a local live board |
| 2 | Intent target | Which kind of improvement matters most |
| 3 | Success proof | What evidence would prove the goal worked |
| 4 | Scope and non-goals | What must stay untouched |
| 5 | Goal handling | Reuse an existing goal, create fresh, or inspect first |

Each step waits for your answer before asking the next. For a specific, concrete goal, `goal-prep` may skip some steps or proceed directly to board creation.

### 3. Confirm the local board

`goal-prep` starts a local GoalBuddy board for the goal and opens it in the agent's in-app browser at the shared hub `http://goalbuddy.localhost:41737/`. If you selected the default board, the response includes a clickable link:

```text
[Open GoalBuddy board](http://goalbuddy.localhost:41737/<slug>/)
```

The board starts before the task list is fully populated, so cards appear live as `state.yaml` is written.

### 4. Verify created files

`goal-prep` creates exactly:

```text
docs/goals/<slug>/
  goal.md      # charter: objective, oracle, constraints
  state.yaml   # board truth: tasks, receipts, status
  notes/       # overflow findings (populated during /goal runs)
```

It does not create `evidence.jsonl`, `units/`, or `artifacts/`. Those belong to the legacy v1 format.

### 5. Review the printed run command

At the end of the turn, `goal-prep` prints:

```text
/goal Follow docs/goals/<slug>/goal.md.
```

It then asks whether to start now, refine `goal.md`, or stop. Choose based on whether the board shape looks correct.

> **Note:** `goal-prep` never starts `/goal` automatically. It stops at the printed command and waits for your choice.

## Verification

After `goal-prep` completes, confirm the board is healthy:

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

Expected output:

```json
{ "ok": true, "errors": [], "warnings": [] }
```

Common warnings at this stage are a weak `goal.oracle.signal` (placeholder text) and agents in `bundled_not_installed` state. Warnings do not block a `/goal` run.

To check agent availability separately:

```bash
npx goalbuddy doctor --target codex --goal-ready
```

## Troubleshooting

### Board returns 404 after goal-prep opens it

The hub is running but the new goal is not yet registered.

1. Check whether the hub is healthy:

```bash
curl http://127.0.0.1:41737/api/boards
```

2. If that returns GoalBuddy board JSON, register the goal on the same hub:

```bash
npx goalbuddy board docs/goals/<slug>
```

3. If `/api/boards` returns 404 or an unrelated response, a different process owns port 41737. Stop that process, then rerun `npx goalbuddy board docs/goals/<slug>`.

> **Warning:** Do not stop the hub process just because a slug URL returns 404. A 404 on a slug path is normal for an unregistered goal on an otherwise healthy hub.

### Agents show as `bundled_not_installed`

The TOML templates exist but agent configs are not installed to `~/.codex/agents/`. A `/goal` run can continue through PM fallback, but to install dedicated agents:

```bash
npx goalbuddy agents
```

Or if the templates are also missing:

```bash
npx goalbuddy install
```

### `check-goal-state` reports `legacy v1 goal state detected`

The `state.yaml` lacks `version: 2` or contains v1-only fields. Create a fresh goal with the current version of `goal-prep`, or see [Common issues](../troubleshooting/common-issues.md) for the full migration checklist.
