# Run a goal

Run `/goal Follow docs/goals/<slug>/goal.md.` to execute a prepared GoalBuddy board. The PM thread reads the charter and board, works on the active task, writes a receipt, and advances to the next task until a final Judge or PM audit proves the full outcome complete.

## Prerequisites

| Requirement | How to satisfy |
|---|---|
| Prepared goal directory | Complete [Prepare a goal](prepare-a-goal.md) first |
| `goal.md` and `state.yaml` present | Exists at `docs/goals/<slug>/` |
| Exactly one active task in `state.yaml` | `check-goal-state.mjs` reports `ok: true` |
| GoalBuddy version current (recommended) | `npx goalbuddy update` |

## Steps

### 1. Copy the run command from goal-prep output

`goal-prep` printed the exact command at the end of the preparation turn. Use it verbatim, including the trailing period:

```text
/goal Follow docs/goals/<slug>/goal.md.
```

### 2. Understand the PM loop

On each `/goal` turn, the PM thread does the following in order:

1. Reads `goal.md` (charter).
2. Reads `state.yaml` (board truth).
3. Runs `check-goal-state.mjs` when available.
4. Works only on the active task.
5. Assigns the task to Scout, Judge, Worker, or PM based on `type` and `assignee`.
6. Writes a compact receipt to the task in `state.yaml`.
7. Marks the task `done` and selects the next active task.
8. Continues unless a stop condition applies.

`state.yaml` is the source of truth. If `goal.md` and `state.yaml` disagree on task status or active task, `state.yaml` wins.

### 3. Understand the state machine

| Field | Allowed values |
|---|---|
| `goal.status` | `active`, `blocked`, `done` |
| `task.status` | `queued`, `active`, `blocked`, `done` |
| `active_task` | Points to exactly one task `id` |

At any moment, exactly one task is `active`. The PM owns choosing the active task; Scout, Judge, and Worker return receipts but do not select the next task.

### 4. Watch the live board

Open the board at:

```text
http://goalbuddy.localhost:41737/<slug>/
```

Cards move between columns as tasks transition:

| Column | Task status |
|---|---|
| Todo | `queued` |
| In Progress | `active` |
| Blocked | `blocked` |
| Completed | `done` |

### 5. Continue a paused run

If the session ends before the goal completes, restart with the same command:

```text
/goal Follow docs/goals/<slug>/goal.md.
```

The PM re-reads `state.yaml`, finds the current active task, and resumes from there. No re-planning is needed — `state.yaml` holds all prior receipts.

### 6. Handle blocked tasks

A blocked task does not stop the goal. The PM should:

1. Write a blocked receipt on the specific task that cannot proceed.
2. Create or activate the next safe local task.
3. Keep `goal.status: active`.

Set `goal.status: blocked` only when absolutely no safe local work exists at all. Missing credentials, production access, or policy decisions block a specific task, not the whole goal.

### 7. Recognize completion

The goal finishes only when a final Judge or PM audit task produces a receipt with `full_outcome_complete: true`. The audit maps all done task receipts and current verification back to the original goal oracle.

Planning, discovery, a passing small slice, or a clean-looking board is not completion. A queued or active Worker task blocks `goal.status: done`.

## Verification

After each `/goal` turn, confirm the board is consistent:

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

Common errors:

| Error | Fix |
|---|---|
| `Missing version: 2` | Add `version: 2` at the top of `state.yaml` |
| `Not exactly one active task` | Ensure `active_task` points to exactly one task with `status: active` |
| `active_task pointer mismatch` | `active_task` must match the one `active` task id |
| `Active Worker missing allowed_files` | Add `allowed_files`, `verify`, and `stop_if` before activating the Worker |
| `Done tasks missing receipt` | Write a receipt before marking a task `done` |

## Troubleshooting

### `/goal` marks the goal done too early

**Cause:** The goal oracle signal is a placeholder (`<...>`, `tbd`, `unknown`). The PM finds verification clean but has not tested the real outcome.

**Fix:** Set a concrete `goal.oracle.signal` in `state.yaml` before running `/goal`:

```yaml
goal:
  oracle:
    signal: "npm test -- test/billing/router.test.ts exits 0 and all assertions pass"
```

Then run `check-goal-state.mjs` to confirm the warning is resolved.

### `goal.status` is stuck as `blocked`

**Cause:** The whole goal is marked blocked when only one task should be.

**Fix:**
1. Set `goal.status: active` in `state.yaml`.
2. Find the task that cannot proceed and set `status: blocked` with a receipt.
3. Create or activate the next safe local task.

### Board not updating live

**Cause:** The EventSource connection dropped.

**Fix:**
1. Check for the green "Live" dot in the board header.
2. Refresh the browser tab to reconnect.
3. If still frozen, rerun `npx goalbuddy board docs/goals/<slug>`.
