# Agents

GoalBuddy defines four agent roles — Scout, Judge, Worker, and PM — each with a fixed read/write scope, a target reasoning effort, and a required receipt shape. A task card's `type` field determines which role executes it.

## Why they exist as their own concept

Agents are not a standalone primitive in GoalBuddy — `goalbuddy/SKILL.md` is explicit: "Agents are not a separate primitive. They are the assignee type on a task." They are covered here because the per-role contracts (sandbox mode, budget caps, receipt schema, authority boundaries) govern how work happens and what the PM may trust from each returned receipt.

## How agents work

When the PM activates a task, it reads the `type` and `assignee` fields and dispatches accordingly. If dedicated agent configs are installed, Codex or Claude Code spawns the matching configured agent. If they are not installed, the PM thread performs the task directly using the same role contract.

The `goalbuddy prompt` command renders a compact dispatch prompt for the active task:

```bash
npx goalbuddy prompt docs/goals/<slug>
```

For Codex subagent spawning, the `required_spawn_agent_type` is mandatory: use `goal_scout`, `goal_worker`, or `goal_judge` exactly. Do not substitute generic agent names.

## The four roles

### Scout

**Source:** `goalbuddy/agents/goal_scout.toml`, `plugins/goalbuddy/agents/goal-scout.md`

| Attribute | Value |
|---|---|
| Reasoning effort | `low` |
| Sandbox mode | `read-only` |
| Parallel-safe | Yes — may run alongside other Scouts |
| Max shell commands | 12 |
| Max evidence items | 12 |
| Summary limit | 120 words |

Scout is a read-only evidence mapper. Its hard contract:

- Read only. No edits, no staging, no agent spawning, no long-running services.
- Work only on the single active Scout task given by the PM.
- Return evidence, contradictions, and candidate facts. Do not choose the next active task.
- If findings are long, set `note_needed: true` and request a note file path from the PM.

Scout receipt shape (returned as JSON, written to `state.yaml` by PM):

```yaml
receipt:
  result: done
  summary: "Found three high-leverage candidates: flaky auth tests, missing router coverage, stale build docs."
  evidence:
    - test/auth/session.test.ts
    - src/router/index.ts
  spawned_tasks:
    - T004
```

The validator requires `summary` and either `evidence` or `note` on a done Scout receipt.

### Worker

**Source:** `goalbuddy/agents/goal_worker.toml`, `plugins/goalbuddy/agents/goal-worker.md`

| Attribute | Value |
|---|---|
| Reasoning effort | `medium` |
| Sandbox mode | `workspace-write` |
| Parallel-safe | No — never assume parallel safety |
| Max verify attempts | 2 |
| Write scope | Only files matching `allowed_files` |

Worker is a bounded writer. Its hard contract:

- Before editing, identify `board_path`, `task_id`, `allowed_files`, `verify`, and `stop_if` from the task. If any are missing, stop.
- Edit only files matching `allowed_files`. Never edit GoalBuddy control files unless explicitly listed there.
- Complete the whole assigned slice. Do not stop after one subcomponent if remaining subcomponents are inside `allowed_files` and verification is still feasible.
- Run the `verify` commands exactly as listed after edits. At most two fix attempts.
- Stop immediately if a file outside `allowed_files` is needed, sources/tests conflict, or verification fails twice.
- Do not decide whether a phase, risk, ambiguity, or completion review is needed — that is the PM's decision.

Worker receipt shape:

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

> **Note:** All command entries must include `status: pass` for done Worker receipts. The validator errors on any non-`pass` command status.

### Judge

**Source:** `goalbuddy/agents/goal_judge.toml`, `plugins/goalbuddy/agents/goal-judge.md`

| Attribute | Value |
|---|---|
| Reasoning effort | `high` |
| Sandbox mode | `read-only` |
| Parallel-safe | Yes (read-only) |

Judge is a skeptical read-only gate. Its hard contract:

- Read only. No edits, no staging, no implementation.
- Read state receipts before raw files. Then read only the inputs named in the Judge task.
- Be skeptical of progress. Lots of files, docs, or tests do not constitute completion.
- Choose the largest safe useful slice for any recommended Worker task. "Safety does not mean tiny."
- Detect micro-slice loops: reject another tiny helper when the board has enough scaffolding for a vertical slice.
- Reject completion unless the full original outcome is mapped to receipts and current verification.
- Do not generate routine next tasks or choose the active task — the PM owns continuation.

Judge is used for:
- Phase transitions and boundary decisions
- Risky or ambiguous scope
- Parallel-safety analysis
- Final completion audit

Judge is not used for routine post-Worker review. Use Judge only at genuine decision points.

Judge receipt shape:

```yaml
receipt:
  result: done
  decision: approved   # approved | rejected | approve_subgoal | reject_subgoal | not_complete | complete
  full_outcome_complete: false
  rationale: "Router coverage is verified; continue with the next PM-selected work package."
  blocked_tasks:
    - T005
```

### PM

The PM is the main `/goal` thread — not a sub-agent but the orchestrating role.

| Attribute | Value |
|---|---|
| Recommended effort (specific goals) | `medium` |
| Recommended effort (open-ended / recovery / audit) | `high` |
| Recommended effort (high-risk final audit) | `xhigh` (optional) |
| Write scope | Control files: `goal.md`, `state.yaml`, `notes/` |

The PM is the only role that may:

- Choose the active task
- Update the board
- Mark tasks done or blocked
- Mark the goal complete
- Create or update `goal.md`, `state.yaml`, and `notes/`

## Computed gate

Write permissions are computed from the active task type — there is no stored boolean gate. Source: `goalbuddy/SKILL.md` → "Computed Gate":

| Active task type | Edits allowed |
|---|---|
| `scout` | None; receipt must include findings or a note |
| `judge` | None; receipt must include a decision |
| `worker` | Only inside `allowed_files`; receipt must include changed files and commands |
| `pm` | Control files only (`goal.md`, `state.yaml`, `notes/`) |

## Installing agents

Agents ship bundled with GoalBuddy but must be installed before dedicated delegation is available:

```bash
# Install Scout, Worker, and Judge agent configs
npx goalbuddy agents

# Check installation status
npx goalbuddy doctor
```

The `agents` block in `state.yaml` records the result:

```yaml
agents:
  scout: installed
  worker: installed
  judge: installed
```

Agent status values and their meaning:

| Status | Meaning | Next action |
|---|---|---|
| `installed` | Config verified in expected agent location | Continue |
| `bundled_not_installed` | Template exists but no installed config | Run `npx goalbuddy agents` |
| `missing` | Neither installed config nor bundled template found | Run `npx goalbuddy install` |
| `unknown` | Availability not checked | Run `npx goalbuddy doctor` |

When agents are not installed, the PM thread falls back to performing Scout/Judge/Worker-shaped tasks directly using the same role contracts.

## Slice sizing

Judge and PM share responsibility for keeping slice sizes healthy. From `goalbuddy/SKILL.md`: "A good task is the largest safe useful slice. Safe means bounded, explicit, verified, and reversible. It does not mean tiny."

Good Worker tasks: working screen, working API path, data pipeline step, real bug fix, milestone review.

Bad Worker tasks: one more helper, projection function, contract file, read-only proof, doc note — unless that tiny item is truly blocking progress.

`check-goal-state.mjs` warns when (`goalbuddy/scripts/check-goal-state.mjs` → `microSliceWarnings()`):

- Three or more recent Worker tasks look tiny (`isTinyTask()` matches objective text)
- A micro Worker/Judge loop is detected over two or more consecutive pairs
- An active Worker has only 1–2 `allowed_files` after 10 or more completed tasks

## Interaction with other concepts

| Concept | Relationship |
|---|---|
| [The loop](./the-loop.md) | Step 6 of the loop dispatches the active task to the correct agent role. |
| [The board](./the-board.md) | Each task card carries `type` and `assignee`; receipts are written back to the card by the PM after the agent returns. |
| [The oracle](./oracle.md) | Judge is responsible for verifying at final completion that receipts and verification map back to the oracle. |
| [Local board](./local-board.md) | The board viewer shows role badges (`Scout`, `Worker`, `Judge`, `PM`) on each card. |

## Common gotchas

**Worker stopping too early.** The Worker contract requires completing the whole assigned slice. Stopping after one subcomponent when the remaining subcomponents are inside `allowed_files` violates the contract and the checker will flag the receipt.

**PM using Judge after every Worker.** The PM should activate the next Worker directly after a successful package unless a genuine review boundary exists. Automatic Judge-after-Worker creates micro-slice loops.

**Missing `allowed_files` before activating a Worker.** The validator hard-errors on an active Worker without `allowed_files`, `verify`, and `stop_if`. The Judge's recommended Worker task must include all three before the PM activates it.

**Substituting generic agents in Codex dispatch.** When using `goalbuddy prompt` for Codex subagent dispatch, `required_spawn_agent_type` must be `goal_scout`, `goal_worker`, or `goal_judge` — never a generic `scout`, `worker`, or `judge`. Source: `goalbuddy/scripts/render-task-prompt.mjs` → `ROLE_DEFAULTS`.
