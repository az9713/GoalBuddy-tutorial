# CLI reference

All `npx goalbuddy` commands. The binary name is `goalbuddy` (also aliased as `goal-maker` for compatibility). Requires Node.js >= 18.

---

## `npx goalbuddy` (bare invocation)

Installs or updates the GoalBuddy skill and agent configs.

```bash
npx goalbuddy
```

Installs the skill payload into Codex and Claude Code, and writes agent configs:

```text
~/.codex/plugins/cache/goalbuddy/goalbuddy/<version>/
~/.codex/agents/goal_scout.toml
~/.codex/agents/goal_worker.toml
~/.codex/agents/goal_judge.toml
```

Source: `internal/cli/goal-maker.mjs`, `internal/cli/postinstall.mjs`

---

## `npx goalbuddy board <goal-dir>`

Starts a local board server for a goal, or registers the goal with an existing hub on port 41737.

```bash
npx goalbuddy board docs/goals/<slug>
npx goalbuddy board docs/goals/<slug> --once --json
```

**Options:**

| Flag | Default | Description |
|---|---|---|
| `--goal <path>` | (required) | Goal directory containing `state.yaml`. Positional arg also accepted. |
| `--host <host>` | `127.0.0.1` | Bind host. Also sets the public hostname for printed URLs. |
| `--port <port>` | `41737` | Server port. |
| `--once` | off | Generate `.goalbuddy-board/` files and exit without starting a server. |
| `--json` | off | Print structured JSON output. |

**Normal output (first board = new hub):**

```text
GoalBuddy local board: http://goalbuddy.localhost:41737/<slug>/
GoalBuddy local hub: http://goalbuddy.localhost:41737/
Watching: docs/goals/<slug>/state.yaml
Press Ctrl-C to stop.
```

**Output when registering with an existing hub:**

```text
GoalBuddy local board: http://goalbuddy.localhost:41737/<slug>/
GoalBuddy local hub: http://goalbuddy.localhost:41737/
Registered with the existing GoalBuddy local board hub.
```

**Hub API endpoints:**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/boards` | List all registered boards |
| `POST` | `/api/boards` | Register a board. Body: `{ "goalDir": "<path>" }` |
| `GET` | `/api/settings` | Read viewer settings |
| `PUT` | `/api/settings` | Save viewer settings |
| `GET` | `/<slug>/api/board` | Current board payload JSON |
| `GET` | `/<slug>/events` | SSE stream. Event: `board`, data: board payload |
| `GET` | `/` | Redirect to preferred board |
| `GET` | `/<slug>/` | Serve the static board viewer |

Source: `goalbuddy/surfaces/local-goal-board/scripts/local-goal-board.mjs`

---

## `npx goalbuddy prompt <goal-dir>`

Renders a compact task prompt for the active task (or a named task). Used when dispatching agents to native Codex or Claude Code agent flows.

```bash
npx goalbuddy prompt docs/goals/<slug>
npx goalbuddy prompt docs/goals/<slug> --task T003
npx goalbuddy prompt docs/goals/<slug> --json
```

**Options:**

| Flag | Description |
|---|---|
| `--task <id>` | Render prompt for a specific task id instead of `active_task` |
| `--board <path>` | Path to `state.yaml` directly |
| `--json` | Print structured JSON output |

**Key output fields:**

| Field | Description |
|---|---|
| `metadata.required_spawn_agent_type` | Agent type to use: `goal_scout`, `goal_worker`, `goal_judge`, or `null` (PM) |
| `metadata.recommended_reasoning` | `low`, `medium`, `high`, or `xhigh` |
| `metadata.sandbox` | `read-only` or `workspace-write` |
| `metadata.fork_context_allowed` | `true` for Scout/Judge, `false` for Worker |
| `metadata.warnings` | Issues found with this task or oracle |
| `task` | Full task fields from `state.yaml` |
| `receipt_schema` | JSON shape the agent must return |

> **Note:** When dispatching Codex subagents, use `required_spawn_agent_type` exactly (`goal_scout`, `goal_worker`, `goal_judge`). Do not substitute generic agent type names.

Source: `goalbuddy/scripts/render-task-prompt.mjs`

---

## `npx goalbuddy parallel-plan <goal-dir>`

Inspects active tasks across a board and its child subgoal boards for parallel safety. Read-only — does not mutate `state.yaml`, spawn agents, or create sub-goals.

```bash
npx goalbuddy parallel-plan docs/goals/<slug>
npx goalbuddy parallel-plan docs/goals/<slug> --json
```

**Options:**

| Flag | Description |
|---|---|
| `--board <path>` | Path to `state.yaml` directly |
| `--json` | Print structured JSON output |

**Output per task candidate:**

| Field | Description |
|---|---|
| `board_path` | Absolute path to the task's `state.yaml` |
| `task_id` | Task id (e.g. `T003`) |
| `role` | `scout`, `judge`, `worker`, or `pm` |
| `recommended_agent` | `goal_scout`, `goal_worker`, `goal_judge`, or `PM` |
| `reasoning_hint` | Recommended reasoning level |
| `safe_to_parallelize` | `true` or `false` |
| `reason` | Why the task is or is not parallel-safe |
| `render_prompt_command` | Command to get the dispatch prompt for this task |

**Parallel safety rules:**
- Scout and Judge: always `true` (read-only)
- Worker: `true` only when `allowed_files` patterns are provably disjoint across all active Workers
- PM: never parallel-safe (mutates board truth)

Source: `goalbuddy/scripts/parallel-plan.mjs`

---

## `npx goalbuddy doctor`

Checks GoalBuddy install health.

```bash
npx goalbuddy doctor
npx goalbuddy doctor --target codex --goal-ready
npx goalbuddy doctor --target claude
```

**Options:**

| Flag | Description |
|---|---|
| `--target codex` | Check Codex install |
| `--target claude` | Check Claude Code install |
| `--goal-ready` | Verify agent configs are in place for a goal run |

---

## `npx goalbuddy update`

Updates GoalBuddy for both Codex and Claude Code to the latest npm version.

```bash
npx goalbuddy update
```

---

## `npx goalbuddy agents`

Installs Scout, Worker, and Judge TOML agent configs to `~/.codex/agents/`. Run when agents show as `bundled_not_installed`.

```bash
npx goalbuddy agents
```

---

## `npx goalbuddy install`

Full install. Use when `npx goalbuddy agents` reports missing templates.

```bash
npx goalbuddy install
```

---

## `node check-goal-state.mjs <state-path>`

Validates a `state.yaml` file against all GoalBuddy v2 rules. Not an `npx goalbuddy` subcommand — run directly via Node.

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

**Options:**

| Flag | Description |
|---|---|
| `--child` | Validate as a subgoal (disallows nested `subgoal:` blocks) |

**Output:**

```json
{
  "ok": true,
  "version": 2,
  "state_path": "docs/goals/my-goal/state.yaml",
  "goal_status": "active",
  "active_task": "T001",
  "agent_statuses": { "scout": "unknown", "worker": "unknown", "judge": "unknown" },
  "task_count": 4,
  "errors": [],
  "warnings": []
}
```

**Exit codes:** `0` = no errors, `1` = one or more errors.

**Common hard errors:**

| Error | Cause |
|---|---|
| `state.yaml must declare version: 2` | Missing or wrong `version` field |
| `goal.status must be active, blocked, or done` | Invalid `goal.status` value |
| `exactly one active task is required` | Zero or multiple tasks have `status: active` |
| `active_task must point to active task <id>` | `active_task` and the actual active task id disagree |
| `active Worker task <id> must include allowed_files` | Worker activated without required fields |
| `done task <id> missing receipt` | Task marked done without a receipt |
| `Worker receipt for <id> has non-passing command status` | A command entry has `status: fail` |
| `done goals require concrete completion proof` | Oracle fields are placeholder text at completion |

Source: `goalbuddy/scripts/check-goal-state.mjs`
