# Write a receipt

A receipt is the compact durable result written to a task card in `state.yaml` when a task completes, blocks, or escalates. Write one for every task before marking it `done` or `blocked`.

## Prerequisites

| Requirement | How to satisfy |
|---|---|
| Active GoalBuddy board | Complete [Prepare a goal](prepare-a-goal.md) |
| `/goal` run in progress | At least one task has `status: active` |
| Task type known | `type` field on the task card: `scout`, `judge`, `worker`, or `pm` |

## Steps

### 1. Identify the task type

Open `docs/goals/<slug>/state.yaml` and find the active task. The `type` field determines the receipt shape.

| Type | Write access | Receipt must include |
|---|---|---|
| `scout` | None | `summary`, `evidence` or `note` |
| `judge` | None | `decision`, `full_outcome_complete`, `rationale` |
| `worker` | Inside `allowed_files` only | `changed_files`, `commands` (with `status: pass`), `summary` |
| `pm` | Control files only | `summary` |

### 2. Write the receipt for the task type

#### Scout receipt

```yaml
receipt:
  result: done
  summary: "Found three high-leverage candidates: flaky auth tests, missing router coverage, stale build docs."
  evidence:
    - test/auth/session.test.ts
    - src/router/index.ts
    - README.md
  spawned_tasks:
    - T004
```

`summary` must be 120 words or fewer. `evidence` lists file paths or source references backing the finding. Include `spawned_tasks` only when Scout findings caused new task IDs to be added to the board.

#### Judge receipt

```yaml
receipt:
  result: done
  decision: "approved"
  full_outcome_complete: false
  rationale: "Router coverage is verified; continue with the next PM-selected work package."
  blocked_tasks:
    - T005
```

`decision` is one of: `approved`, `rejected`, `approve_subgoal`, `reject_subgoal`, `not_complete`, `complete`.

`full_outcome_complete` must be `true` only on the final audit task when the complete original outcome is mapped to receipts and current verification. Do not set it `true` after a single passing work package.

#### Worker receipt

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
  summary: "invoice.paid now routes through eventRouter.dispatch; regression test added."
```

`changed_files` must list every file edited. Every file listed must match a pattern in the task's `allowed_files`. Every `commands` entry must show `status: pass`.

#### Blocked receipt (any task type)

```yaml
receipt:
  result: blocked
  summary: "Need credentials for production database before T007 can run verify."
  remaining_blockers:
    - "PROD_DB_URL env var required"
```

### 3. Write a note file for long findings

When Scout, Judge, or PM output is too large for an inline receipt, write a note file and point the receipt to it.

Note file path: `docs/goals/<slug>/notes/<task-id>-<slug>.md`

Note file format (defined in `goalbuddy/templates/note.md`):

```markdown
# <Task ID>: <Note Title>

Task: `<T###>`
Kind: `scout | judge | pm`
Status: `current | superseded | blocked`

## Summary

<One paragraph the board receipt can point to.>

## Details

- <Evidence, decision, or context too large for an inline receipt.>
```

The receipt in `state.yaml` then becomes:

```yaml
receipt:
  result: done
  note: notes/T001-repo-map.md
  summary: "Repo map completed; three candidate tranches found."
```

### 4. Mark the task done and update the board

After writing the receipt, set `task.status: done`. The PM then updates `active_task` to point to the next task's `id` and sets that task's `status: active`.

> **Note:** Only the PM owns this transition. Scout, Judge, and Worker return receipts; they do not choose the next active task or mark the goal complete.

### 5. Run the validator

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

Expected output after a valid receipt:

```json
{ "ok": true, "errors": [], "warnings": [] }
```

## Verification

Common validator errors related to receipts:

| Error | Cause | Fix |
|---|---|---|
| `Done tasks missing receipt` | Task has `status: done` but no `receipt` field | Add the receipt before marking done |
| `Done Worker receipts with non-passing command statuses` | A `commands` entry has `status: fail` | Fix the verify command and update the status |
| `changed_files includes a file not matching allowed_files` | Worker edited a file outside the approved scope | Add the file to `allowed_files` before running the Worker |
| `Active Worker missing allowed_files` | Worker activated without `allowed_files` | Add `allowed_files`, `verify`, and `stop_if` |

## Troubleshooting

### Worker receipt fails `check-goal-state` on `changed_files`

**Cause:** `changed_files` lists a file that does not match any pattern in the task's `allowed_files`.

**Fix:** Before running the Worker again, add the file pattern to `allowed_files` in `state.yaml`. If the receipt was already written, either remove the out-of-scope file from `changed_files` or expand `allowed_files` and re-run the check.

### Receipt has commands with `status: fail`

**Cause:** A verify command reported a failure. Done Worker receipts must have all commands at `status: pass`.

**Fix:** Re-run the failing command, fix the underlying issue, and update the status entry to `pass`. If the failure is a known-flaky test, record the rationale in `summary` and escalate to a Judge task to make an explicit accept/reject decision.
