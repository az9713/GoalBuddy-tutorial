# state.yaml reference

`state.yaml` is the machine truth for a GoalBuddy goal. Every task, receipt, agent status, oracle, and completion decision lives here. The file is created by `goal-prep`, updated by the PM on every loop turn, and validated by `check-goal-state.mjs`.

> **Note:** When `goal.md` and `state.yaml` disagree on task status, active task, or completion, `state.yaml` wins.

---

## Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | integer | Yes | Must be `2`. The validator hard-errors on any other value. |
| `active_task` | string | Yes (when active) | Id of the one active task. `null` when `goal.status: done`. |
| `goal` | mapping | Yes | Goal metadata. See [goal block](#goal-block). |
| `rules` | mapping | Yes | Operating constraints. See [rules block](#rules-block). |
| `agents` | mapping | Yes | Agent availability status. See [agents block](#agents-block). |
| `visual_board` | mapping | Yes | Local board state. See [visual_board block](#visual_board-block). |
| `tasks` | sequence | Yes | Task cards. At least one required. See [tasks](#tasks). |
| `checks` | mapping | No | Verification freshness state. See [checks block](#checks-block). |

---

## `goal` block

```yaml
goal:
  title: "Add OAuth to the API"
  slug: "add-oauth-api"
  kind: specific
  tranche: "Implement GitHub OAuth callback and wire into auth router."
  status: active
  oracle:
    signal: "npm test -- test/auth/ exits 0 and all assertions pass"
    cadence: "after each Worker package and at final audit"
    final_proof: "T003 Worker receipt + passing test run"
  intake:
    original_request: "Add OAuth login to our Express API."
    interpreted_outcome: "Users can log in via GitHub OAuth and receive a JWT session cookie."
    input_shape: specific
    audience: "API users"
    authority: approved
    proof_type: test
    completion_proof: "npm test -- test/auth/ exits 0; manual walkthrough shows JWT set in session"
    likely_misfire: "Implementing OAuth UI instead of backend callback handler"
    blind_spots_considered:
      - "Session storage compatibility"
    existing_plan_facts: []
```

| Field | Type | Values | Description |
|---|---|---|---|
| `title` | string | Any | Human-readable goal name. Shown in the local board header. |
| `slug` | string | `[a-z0-9-]+` | URL-safe identifier. Used as the board URL path and goal directory name. |
| `kind` | enum | `specific`, `open_ended`, `existing_plan`, `recovery`, `audit` | Goal classification. Affects default seed board shape. |
| `tranche` | string | Any | Scope of the current execution run. Shown in the board tranche line. |
| `status` | enum | `active`, `blocked`, `done` | Goal-level status. Continuous goals must stay `active` until full outcome is proven. |

### `goal.oracle` sub-block

| Field | Type | Required | Description |
|---|---|---|---|
| `signal` | string | Yes (not placeholder) | Live check applied after each Worker package and at final audit. A placeholder value triggers a warning during runs and a hard error at completion. |
| `cadence` | string | No | When the oracle is checked. |
| `final_proof` | string | Yes (not placeholder) | Receipt-backed evidence required before `full_outcome_complete: true` can be recorded. |

Placeholder values (`<...>`, `unknown`, `tbd`, `todo`, `none`, empty) are detected by `isWeakProof()` in `check-goal-state.mjs` and produce warnings or errors.

### `goal.intake` sub-block

| Field | Type | Description |
|---|---|---|
| `original_request` | string | Shortest faithful copy of what the user asked for. |
| `interpreted_outcome` | string | One sentence: what must become true. |
| `input_shape` | enum | `vague`, `specific`, `existing_plan`, `recovery`, `audit` |
| `audience` | string | Beneficiary of the outcome. |
| `authority` | enum | `requested`, `approved`, `inferred`, `needs_approval`, `blocked` |
| `proof_type` | enum | `test`, `demo`, `artifact`, `metric`, `review`, `source_backed_answer`, `decision` |
| `completion_proof` | string | Observable signal that closes the full original outcome. |
| `likely_misfire` | string | How `/goal` could succeed at the wrong thing. |
| `blind_spots_considered` | sequence | Risks or unstated choices surfaced during diagnostic intake. |
| `existing_plan_facts` | sequence | User-provided steps, files, constraints, or sequencing to preserve and validate. |

---

## `rules` block

Rules encode operating constraints. The checker reads these directly from YAML text. All default to their documented values when present in the template.

| Field | Default | Description |
|---|---|---|
| `pm_owns_state` | `true` | Only the PM may choose the active task, update the board, or mark completion. |
| `one_active_task` | `true` | Exactly one task may have `status: active` at any time. |
| `max_write_workers` | `1` | At most one write-capable Worker active simultaneously. |
| `no_implementation_without_worker_or_pm_task` | `true` | No code edits outside an active Worker or PM task. |
| `no_completion_without_judge_or_pm_audit` | `true` | The goal cannot close without a final Judge or PM audit receipt. |
| `planning_is_not_completion` | `true` | Planning, discovery, and task selection do not satisfy the oracle. |
| `queued_required_worker_blocks_completion` | `true` | A queued Worker that is still required blocks `goal.status: done`. |
| `continuous_until_full_outcome` | `true` | The PM continues until `full_outcome_complete: true` is recorded. |
| `missing_input_or_credentials_do_not_stop_goal` | `true` | Missing credentials block a specific task, not the whole goal. |
| `preserve_and_validate_existing_plan` | `true` | User-supplied plans are preserved and validated before execution. |
| `intake_misfire_must_be_audited` | `true` | The `likely_misfire` field is re-checked on every loop turn. |
| `goal_pressure_requires_oracle` | `true` | Weak oracle fields produce warnings. |
| `no_completion_on_weak_proof` | `true` | Weak oracle fields produce hard errors at `goal.status: done`. |

### `rules.slice_policy` sub-block

| Field | Default | Description |
|---|---|---|
| `max_consecutive_tiny_tasks` | `2` | Triggers a warning after this many consecutive tiny Worker tasks. |
| `prefer_vertical_slices` | `true` | Judge should prefer vertical slices over helpers/contracts. |
| `judge_picks_largest_safe_slice` | `true` | Judge selects the largest safe useful slice. |
| `worker_completes_whole_slice` | `true` | Worker completes the full assigned slice, not just one subcomponent. |

---

## `agents` block

| Field | Values | Description |
|---|---|---|
| `scout` | `installed`, `bundled_not_installed`, `missing`, `unknown` | Whether `goal_scout.toml` is installed at the expected user or project agent location. |
| `worker` | same | Whether `goal_worker.toml` is installed. |
| `judge` | same | Whether `goal_judge.toml` is installed. |

| Status | Meaning | Next action |
|---|---|---|
| `installed` | Config verified in expected location | Continue |
| `bundled_not_installed` | Template exists but not installed | Run `npx goalbuddy agents` |
| `missing` | Neither template nor installed config found | Run `npx goalbuddy install` |
| `unknown` | Availability not checked | Run `npx goalbuddy doctor` |

All non-`installed` statuses produce warnings, not hard errors. The PM thread falls back to performing tasks directly when dedicated agents are unavailable.

---

## `visual_board` block

| Field | Values | Description |
|---|---|---|
| `selected` | `none`, `local`, `unknown` | Which board surface is in use. |
| `local.status` | `not_requested`, `starting`, `live`, `generated`, `blocked` | Local board server state. |
| `local.url` | URL or `null` | Board URL, e.g. `http://goalbuddy.localhost:41737/my-goal/` |
| `local.command` | string | Command to start the board: `npx goalbuddy board docs/goals/<slug>` |

---

## `tasks`

A YAML sequence of task cards. At least one task is required. Tasks are ordered by id.

### Task card fields

| Field | Type | Required | Values | Description |
|---|---|---|---|---|
| `id` | string | Yes | `T001`–`T999` | Unique task identifier. `T###` format enforced by validator. |
| `type` | enum | Yes | `scout`, `judge`, `worker`, `pm` | Determines agent role and write permissions. |
| `assignee` | enum | Yes | `Scout`, `Judge`, `Worker`, `PM` | Must match `type`. |
| `status` | enum | Yes | `queued`, `active`, `blocked`, `done` | Task lifecycle state. |
| `reasoning_hint` | enum | No | `default`, `low`, `medium`, `high`, `xhigh` | PM guidance to the agent's reasoning effort. |
| `objective` | string | Yes | Any | One sentence describing the task goal. |
| `inputs` | sequence | No | File paths, task receipt references | What the agent should read first. |
| `constraints` | sequence | No | Natural language | Limits on what the task may do. |
| `expected_output` | sequence | No | Natural language | What a valid receipt must demonstrate. |
| `receipt` | mapping or null | No | See receipts | Durable result written when task completes/blocks. |

### Additional fields for Worker tasks

| Field | Required when active | Description |
|---|---|---|
| `allowed_files` | Yes | Glob patterns for files the Worker may edit. The validator hard-errors if missing on an active Worker. |
| `verify` | Yes | Commands to run after edits. Must include at least one. |
| `stop_if` | Yes | Conditions that stop execution (e.g. "Need files outside allowed_files"). |

### Optional subgoal link (any task type)

| Field | Required | Description |
|---|---|---|
| `subgoal.status` | Yes | `active`, `blocked`, or `done` |
| `subgoal.path` | Yes | Relative path to child `state.yaml`, e.g. `subgoals/T003-child/state.yaml`. Must be inside goal root. |
| `subgoal.owner` | No | Agent responsible for the child board. |
| `subgoal.created_from` | No | Parent task id. |
| `subgoal.depth` | Yes | Must be `1`. Nesting beyond depth 1 is not supported. |
| `subgoal.rollup_receipt` | No | Summary of the child board's outcome, populated when it closes. |

---

## Receipt shapes

All receipts include `result: done | blocked`.

### Scout receipt

```yaml
receipt:
  result: done
  summary: "Found three high-leverage candidates."   # <= 120 words
  evidence:
    - test/auth/session.test.ts
    - src/router/index.ts
  facts:
    - "passport.js is already a dependency but unused"
  contradictions: []
  ambiguity_requiring_judge: []
  note: notes/T001-repo-map.md   # optional: for long findings
  spawned_tasks:
    - T004
```

Validator requires: `summary` AND (`evidence` OR `note`) on a done Scout receipt.

### Judge receipt

```yaml
receipt:
  result: done
  decision: approved   # approved | rejected | approve_subgoal | reject_subgoal | not_complete | complete
  full_outcome_complete: false
  rationale: "Router coverage verified; continue."   # <= 120 words
  evidence:
    - src/router/index.ts
  blocked_tasks:
    - T005
  missing_evidence: []
  required_board_updates: []
```

Validator requires: `decision` on a done Judge receipt. For a final closing audit, `decision: complete` AND `full_outcome_complete: true` are both required.

### Worker receipt

```yaml
receipt:
  result: done
  changed_files:
    - src/billing/router.ts
    - test/billing/router.test.ts
  commands:
    - cmd: git diff --check
      status: pass
    - cmd: npm test -- test/billing/router.test.ts
      status: pass
  summary: "invoice.paid now routes through eventRouter.dispatch."   # <= 120 words
  remaining_blockers: []
  verification_attempts: 1
  stopped_because: null
```

Validator requires on a done Worker receipt: `changed_files` (at least one), `commands` (with all `status: pass`), `summary`. Every file in `changed_files` must match a pattern in the task's `allowed_files`.

### Blocked receipt (any type)

```yaml
receipt:
  result: blocked
  summary: "Need PROD_DB_URL before T007 can run verify."
  remaining_blockers:
    - "PROD_DB_URL env var required"
```

Validator requires: a receipt with `result: blocked` on any task with `status: blocked`.

---

## `checks` block

| Field | Values | Description |
|---|---|---|
| `dirty_fingerprint` | string or `unknown` | Hash of repo state at last verification. Used to detect stale verification. |
| `last_verification.result` | `pass`, `fail`, `unknown` | Outcome of the most recent verification run. |
| `last_verification.task` | string or `null` | Task id that ran the last verification. |
| `last_verification.commands` | sequence | Commands run during last verification. |

---

## Allowed root entries

The goal root (`docs/goals/<slug>/`) may contain only:

```text
goal.md
state.yaml
notes/
subgoals/
.goalbuddy-board/
```

The validator errors on any other file or directory. Source: `check-goal-state.mjs` → `rootEntryErrors()`.
