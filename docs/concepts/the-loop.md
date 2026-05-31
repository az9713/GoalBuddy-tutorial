# The loop

Every `/goal` continuation follows a fixed 12-step PM loop that runs from charter read to receipt write. The loop is the engine that keeps a long-running goal advancing without human decomposition at each step.

## Why it exists as its own concept

A multi-hour coding goal with Scout, Judge, and Worker delegation can sprawl unless every continuation begins from the same orientation point. The loop enforces that orientation. It also prevents the most common failure modes: stopping prematurely at "ready for implementation," reviewing every tiny Worker by habit, and losing sight of the original user outcome amid task receipts.

## How it works

On every `/goal Follow docs/goals/<slug>/goal.md.` invocation, the PM thread runs these steps in order:

1. Read `goal.md` (the charter).
2. Read `state.yaml` (board truth).
3. Run the bundled GoalBuddy update checker when available and mention a newer version without blocking.
4. Re-check the intake: original request, input shape, authority, proof, blind spots, existing plan facts, and likely misfire.
5. Work only on the active board task.
6. Assign Scout, Judge, Worker, or PM according to the task type.
7. Write a compact task receipt.
8. Update the board.
9. If safe local work remains, choose the next largest reversible Worker package and continue unless blocked.
10. If a problem, suggestion, or follow-up should become a repo artifact, create an approved issue/PR or ask the operator whether to create one.
11. Review at phase, risk, rejected-verification, ambiguity, or final-completion boundaries only — do not review every small Worker by habit.
12. Finish only with a Judge/PM audit receipt that maps receipts and verification back to the original user outcome and records `full_outcome_complete: true`.

The loop is defined in `goalbuddy/templates/goal.md` (the PM Loop section) and reflects the invariant stated in `goalbuddy/SKILL.md`:

```text
Intent -> Oracle -> Surface -> Loop -> Proof
```

### The default run is continuous

Unless the user explicitly asks for planning only, the loop does not stop after Scout findings, Judge decisions, or a single verified Worker package. The default continuation pattern is:

```text
Discover enough evidence -> choose the largest reversible local work package ->
implement it -> verify it -> review only at risk or phase boundaries ->
immediately choose and execute the next work package ->
repeat until the full original outcome is complete
```

### The update check

Step 3 runs the bundled checker without blocking progress:

```bash
node <skill-path>/scripts/check-update.mjs --json
```

If `update_available: true` is returned, the PM mentions the newer version once and continues.

### The re-check of intake (step 4)

Step 4 is not ceremony. Re-reading the intake fields — `original_request`, `likely_misfire`, `authority`, `completion_proof` — is how the PM avoids completing the wrong goal. An intake field called `likely_misfire` exists precisely to surface the most common way a `/goal` run could succeed at the wrong thing. Step 4 applies that pressure every time.

### The continuation rule (step 9)

After a task completes, the PM immediately writes its receipt and selects the next active task unless a final audit proves the full original owner outcome is complete. The only legitimate stopping conditions are:

- A final Judge or PM audit receipt with `full_outcome_complete: true`.
- A blocker that cannot be worked around locally (credentials, destructive-operation permission, policy decision) — in which case the specific task is blocked and local safe work continues on another task.

"Ready for implementation" is not a stopping condition. A passed slice is not a stopping condition. A clean-looking board is not a stopping condition.

## Key data structures and their meaning

The loop's state lives in two files:

| File | Role |
|---|---|
| `goal.md` | Human-editable charter. Defines the oracle, non-negotiable constraints, tranche scope, and stop rule. |
| `state.yaml` | Machine truth. Holds the task list, active task pointer, receipts, agent statuses, and verification freshness. When the two disagree, `state.yaml` wins. |

The loop reads both on every continuation. It writes only to `state.yaml` (and `notes/` for long findings).

## Interaction with other concepts

| Concept | Relationship |
|---|---|
| [The board](./the-board.md) | The loop reads and updates the board every turn. The active task pointer determines what work is allowed. |
| [The oracle](./oracle.md) | Step 4 re-checks the oracle signal. Step 12 requires the final audit to map back to it. |
| [Agents](./agents.md) | Step 6 dispatches the active task to Scout, Judge, Worker, or PM. Each agent returns a receipt that the loop writes in step 7. |
| [Local board](./local-board.md) | The local board server watches `state.yaml` and `notes/` for changes; each board update written by the loop propagates live to any open viewer. |

## Common gotchas

**Stopping after one package.** The loop's step 9 explicitly says to advance to the next work package after a verified Worker completes. A completed slice does not close the goal.

**Reviewing every Worker.** Step 11 says to review at phase, risk, rejected-verification, ambiguity, or final-completion boundaries — not after every Worker by default. A Judge after every tiny Worker is a micro-slice loop, which `check-goal-state.mjs` will flag with a warning.

**Blocked does not mean stop.** When a task hits a blocker, the PM blocks that specific task with a receipt, creates a safe follow-up task, and re-enters the loop. Setting `goal.status: blocked` for missing credentials is a validator error: continuous goals keep `goal.status: active` and block only the affected task.

**Charter vs. board disagreement.** If `goal.md` and `state.yaml` describe different active tasks or different completion states, the loop follows `state.yaml`. The charter is the intent; the board is machine truth.

**`full_outcome_complete: true` is required.** A Judge receipt with `decision: complete` but `full_outcome_complete: false` does not close the goal. Both fields must be present and correct together.
