# Use parallel agents

Run `goalbuddy parallel-plan` to identify which tasks in a GoalBuddy board are safe to hand off to parallel agents. Use it when a `/goal` run has multiple independent Scout or Judge tasks, or Worker tasks with provably disjoint write scopes.

> **Warning:** `goalbuddy parallel-plan` is read-only. It does not mutate `state.yaml`, spawn agents, or create sub-goals. It produces recommendations only.

## Prerequisites

| Requirement | How to satisfy |
|---|---|
| Active GoalBuddy board | Complete [Prepare a goal](prepare-a-goal.md) |
| Board has multiple queued tasks | At least two tasks in `state.yaml` with `status: queued` |
| GoalBuddy installed | `npx goalbuddy` |

## Steps

### 1. Run `parallel-plan`

```bash
npx goalbuddy parallel-plan docs/goals/<slug>
```

The command reads `state.yaml` (and any child board `state.yaml` files linked via `subgoal.path`) and analyzes each task for parallel safety. No files are changed.

### 2. Review the output

`parallel-plan` (`goalbuddy/scripts/parallel-plan.mjs`) produces a per-task analysis. Each entry includes:

| Field | Meaning |
|---|---|
| `role` | The task type: `scout`, `judge`, `worker`, or `pm` |
| `safe_to_parallelize` | `true` or `false` |
| `reason` | Why the task is or is not parallel-safe |
| `render_prompt_command` | The exact command to render a dispatch prompt for this task |

**Scout and Judge tasks are always `safe_to_parallelize: true`** because they are read-only. Worker tasks are parallel-safe only when their `allowed_files` are provably disjoint from all other active Workers.

### 3. Render a dispatch prompt for each safe task

For each task marked `safe_to_parallelize: true`, render a compact dispatch prompt:

```bash
npx goalbuddy prompt docs/goals/<slug>
npx goalbuddy prompt docs/goals/<slug> --task T003
npx goalbuddy prompt docs/goals/<slug> --json
```

The prompt output (`goalbuddy/scripts/render-task-prompt.mjs`) includes:

| Field | Meaning |
|---|---|
| `metadata.required_spawn_agent_type` | GoalBuddy agent type: `goal_scout`, `goal_worker`, or `goal_judge` |
| `task` | Full task fields from `state.yaml` |
| `receipt_schema` | The JSON shape the agent must return |
| `metadata.warnings` | Any issues found with this task |

### 4. Dispatch using the correct agent type

When dispatching Codex subagents, use `required_spawn_agent_type` exactly:

| Prompt value | Correct `spawn_agent` type |
|---|---|
| `goal_scout` | `goal_scout` |
| `goal_worker` | `goal_worker` |
| `goal_judge` | `goal_judge` |

> **Warning:** Do not substitute generic `scout`, `worker`, or `judge` agent types. If the required GoalBuddy agent is unavailable, stop spawning and run `npx goalbuddy agents` or continue as PM fallback.

### 5. Handle `wait_agent` timeouts

After dispatching a subagent, wait for it to return a receipt. If the `wait_agent` call times out with no changes to the task's `allowed_files`:

1. Stop waiting — do not retry with another `wait_agent` call.
2. Record the timeout in the task receipt as `result: blocked` with `remaining_blockers` describing the timeout.
3. Recover: the PM takes over the task or activates a safe local alternative.

One timeout is the limit.

### 6. Apply receipts and advance the board

After each parallel agent returns a receipt, the PM applies it to `state.yaml` and selects the next active task. The one-active-task rule still applies to write-capable Workers: at most one Worker may be active at a time unless `parallel-plan` has confirmed disjoint write scopes.

Run the checker after applying receipts:

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

## Verification

Expected after successful parallel work:

```json
{ "ok": true, "errors": [], "warnings": [] }
```

Check especially for:
- Not exactly one active task (parallel Workers that both wrote state)
- `active_task` pointer mismatch
- Done tasks missing receipt

## Troubleshooting

### `parallel-plan` marks a Worker task `safe_to_parallelize: false` unexpectedly

**Cause:** The task's `allowed_files` overlap with another active Worker's `allowed_files`, or `allowed_files` is empty (unknown write scope).

**Fix:** Narrow `allowed_files` so the two Workers share no file patterns. Run `parallel-plan` again to confirm. If narrowing is not possible, run the Workers sequentially.

### `required_spawn_agent_type` shows `goal_worker` but agent is unavailable

**Cause:** The GoalBuddy Worker agent TOML is not installed.

**Fix:**

```bash
npx goalbuddy agents
```

If that does not resolve it:

```bash
npx goalbuddy install
```

If the agent remains unavailable, continue as PM fallback: the PM thread handles the Worker task directly.
