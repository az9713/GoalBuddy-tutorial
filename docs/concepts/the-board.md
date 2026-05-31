# The board

The board is `docs/goals/<slug>/state.yaml` — the single source of machine truth for a GoalBuddy goal. Every task, receipt, agent status, active task pointer, verification result, and completion decision lives here.

## Why it exists as its own concept

`goal.md` is human-editable and expresses intent; `state.yaml` is machine-writable and expresses current state. Keeping them separate means the PM loop can update facts (receipts, task status, active task pointer) without overwriting the user's original framing. When the two disagree, `state.yaml` wins for all task-level truth.

## How it works

The board is a YAML document with a fixed structure. It is created by `$goal-prep` / `/goal-prep`, updated by the PM thread on every loop turn, and validated by `check-goal-state.mjs`.

### Version requirement

Every v2 board must declare:

```yaml
version: 2
```

The validator hard-errors on missing or wrong versions. Legacy v1 signals (the `gate:`, `artifact_policy:`, `active_unit:`, `units/`, `artifacts/`, and `evidence.jsonl` patterns) produce a migration error.

### Goal metadata

```yaml
goal:
  title: "<Human-readable title>"
  slug: "<url-safe kebab-case identifier>"
  kind: open_ended   # specific | open_ended | existing_plan | recovery | audit
  tranche: "<scope description for this execution run>"
  status: active     # active | blocked | done
  oracle:
    signal: "<observable live check>"
    cadence: "after each Worker package and at final audit"
    final_proof: "<receipt-backed evidence required for completion>"
  intake:
    original_request: "<shortest faithful user request>"
    interpreted_outcome: "<one sentence>"
    input_shape: vague
    audience: unknown
    authority: requested
    proof_type: artifact
    completion_proof: "<observable signal>"
    likely_misfire: "<how /goal could succeed at the wrong thing>"
    blind_spots_considered: []
    existing_plan_facts: []
```

| Kind | When to use |
|---|---|
| `specific` | Concrete outcome, evidence requirements clear |
| `open_ended` | Broad improvement, vague scope |
| `existing_plan` | User supplied a plan to validate then execute |
| `recovery` | Stalled or red board, triage required first |
| `audit` | Read-only analysis; execution only on user approval |

### Rules block

The `rules` block encodes operating constraints that the checker enforces. Defined in `goalbuddy/templates/state.yaml`:

```yaml
rules:
  pm_owns_state: true
  one_active_task: true
  max_write_workers: 1
  no_implementation_without_worker_or_pm_task: true
  no_completion_without_judge_or_pm_audit: true
  planning_is_not_completion: true
  queued_required_worker_blocks_completion: true
  continuous_until_full_outcome: true
  missing_input_or_credentials_do_not_stop_goal: true
  preserve_and_validate_existing_plan: true
  intake_misfire_must_be_audited: true
  goal_pressure_requires_oracle: true
  no_completion_on_weak_proof: true
  slice_policy:
    max_consecutive_tiny_tasks: 2
    prefer_vertical_slices: true
    judge_picks_largest_safe_slice: true
    worker_completes_whole_slice: true
```

Several rules gate hard errors at completion:

- `continuous_until_full_outcome: true` requires `full_outcome_complete: true` in the final audit receipt before `goal.status: done` is accepted.
- `no_completion_on_weak_proof: true` requires concrete `oracle.signal`, `oracle.final_proof`, and `intake.completion_proof` before a goal can close.
- `missing_input_or_credentials_do_not_stop_goal: true` combined with `continuous_until_full_outcome: true` causes a hard error if `goal.status: blocked` is set — those goals must stay `active` and block only affected tasks.

### Agents block

```yaml
agents:
  scout: unknown   # installed | bundled_not_installed | missing | unknown
  worker: unknown
  judge: unknown
```

| Status | Meaning | Effect on /goal |
|---|---|---|
| `installed` | Matching agent config verified in user or project agent location | Full delegation available |
| `bundled_not_installed` | Bundled `.toml`/`.md` template exists but not installed | PM fallback; run `npx goalbuddy agents` to install |
| `missing` | Neither installed config nor bundled template found | PM fallback; run `npx goalbuddy install` |
| `unknown` | Agent availability not checked | PM fallback; run `npx goalbuddy doctor` to check |

Non-`installed` statuses are warnings, not failures. The PM thread can perform Scout/Judge/Worker-shaped tasks directly.

### Visual board block

```yaml
visual_board:
  selected: unknown   # none | local | unknown
  local:
    status: not_requested   # not_requested | starting | live | generated | blocked
    url: null
    command: "npx goalbuddy board docs/goals/<goal-slug>"
```

### Active task pointer

```yaml
active_task: T001
```

`active_task` must point to the one task with `status: active`. The checker validates that exactly one task is active when `goal.status` is `active`, and that `active_task` matches its id.

### Task cards

Each task in the `tasks` list is a card:

```yaml
tasks:
  - id: T001           # T### format required (T001–T999)
    type: scout        # scout | judge | worker | pm
    assignee: Scout    # must match type: Scout | Judge | Worker | PM
    status: queued     # queued | active | blocked | done
    reasoning_hint: default   # default | low | medium | high | xhigh
    objective: "Map repo health and identify improvement candidates."
    inputs: []
    constraints: []
    expected_output: []
    receipt: null
```

Worker tasks additionally require these fields when active:

```yaml
    allowed_files: []   # glob patterns for files the Worker may edit
    verify: []          # commands to run after edits
    stop_if: []         # conditions that stop execution
```

An optional subgoal link can be added to any task:

```yaml
    subgoal:
      status: active
      path: subgoals/T003-child/state.yaml
      owner: Worker
      created_from: T003
      depth: 1
      rollup_receipt: null
```

### Receipts

A receipt is the durable result left on a task card when the task completes, blocks, or escalates. Short findings fit inline; for long findings create `notes/<task-id>-<slug>.md` and point to it:

```yaml
    receipt:
      result: done
      note: notes/T001-repo-map.md
      summary: "Repo map completed; three candidate tranches found."
```

Worker receipts require `changed_files`, `commands`, and `summary`. Each command must record `status: pass`:

```yaml
    receipt:
      result: done
      changed_files:
        - src/billing/router.ts
        - test/billing/router.test.ts
      commands:
        - cmd: npm test -- test/billing/router.test.ts
          status: pass
      summary: "invoice.paid now routes through eventRouter.dispatch."
```

### Checks block

```yaml
checks:
  dirty_fingerprint: unknown
  last_verification:
    result: unknown
    task: null
    commands: []
```

### Allowed root entries

The goal root may contain only:

```text
docs/goals/<slug>/
  goal.md
  state.yaml
  notes/
  subgoals/
  .goalbuddy-board/
```

The validator errors on any other file or directory. Source: `goalbuddy/scripts/check-goal-state.mjs` → `rootEntryErrors()`.

## Validating the board

Run the validator against any `state.yaml`:

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

The validator produces structured JSON and exits 0 on success, 1 on errors:

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
  "warnings": [
    "goal.oracle.signal is missing or placeholder-like; weak oracles make /goal finish too early."
  ]
}
```

Errors are hard failures. Warnings are advisories — the most common ones surface weak oracle fields and agent availability.

## Interaction with other concepts

| Concept | Relationship |
|---|---|
| [The loop](./the-loop.md) | The loop reads the board on every turn and writes receipts, task status, and the active task pointer. |
| [The oracle](./oracle.md) | Oracle fields live inside the board (`goal.oracle.signal`, `goal.oracle.final_proof`). The validator gates completion on them. |
| [Agents](./agents.md) | Each task card names its assignee. Agent availability is recorded in the `agents` block. |
| [Local board](./local-board.md) | The local board server watches `state.yaml` for changes and pushes live updates to the viewer. |

## Common gotchas

**`active_task` mismatch.** If you manually edit `state.yaml` and move a task to `active` without updating `active_task:`, the checker will error. Both must agree.

**Blocking the goal instead of the task.** Setting `goal.status: blocked` on a continuous goal with `missing_input_or_credentials_do_not_stop_goal: true` is a validator error. Block the specific task instead.

**Worker task missing required fields.** An active Worker task without `allowed_files`, `verify`, and `stop_if` is a hard error. The PM must populate these before activating a Worker.

**Done Worker with empty `changed_files`.** The validator requires at least one entry in `changed_files` on a done Worker receipt.

**Subgoal depth.** Only depth 1 is supported. A subgoal whose own `state.yaml` contains a `subgoal:` field will fail the `--child` validation invoked by the parent checker.

**Placeholder oracle fields.** Any value matching `<.*>`, or the strings `unknown`, `tbd`, `todo`, or `none`, is treated as weak. These produce warnings during normal runs and hard errors at completion when `no_completion_on_weak_proof: true`.
