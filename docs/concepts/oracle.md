# The oracle

The oracle is the observable signal that tells the PM whether the original user outcome is actually true yet. Without a concrete oracle, a goal can appear complete while still failing the user's real intent.

## Why it exists as its own concept

Weak proof creates weak goals. Planning, discovery, a passing tiny Worker slice, or a clean-looking board are not evidence that the user's outcome is met. The oracle concept exists to keep continuous pressure on the goal: every loop turn re-checks the intake misfire and oracle signal, and the goal cannot close until a final audit maps receipts and verification directly back to that observable signal.

The `goalbuddy/SKILL.md` invariant makes this explicit:

```text
Intent -> Oracle -> Surface -> Loop -> Proof
```

"No oracle, no serious goal."

## How it works

The oracle is recorded during `$goal-prep` / `/goal-prep` intake. It populates three related fields in `state.yaml` and one field in `goal.md`. All four are checked by `check-goal-state.mjs` at goal completion.

### Three oracle fields in `state.yaml`

```yaml
goal:
  oracle:
    signal: "<live check, walkthrough, artifact, metric, source-backed answer, or decision>"
    cadence: "after each Worker package and at final audit"
    final_proof: "<receipt-backed evidence required before full_outcome_complete: true>"
  intake:
    completion_proof: "<observable signal that proves the full original outcome is complete>"
```

| Field | Purpose |
|---|---|
| `goal.oracle.signal` | The live check the PM applies after each Worker package and at final audit. Must be a concrete observable: a test suite, browser walkthrough, demo transcript, artifact, benchmark, source-backed answer, or formal decision. |
| `goal.oracle.final_proof` | The receipt-backed evidence that must exist before `full_outcome_complete: true` can be recorded. Points to specific receipts, commands, artifacts, or human decisions. |
| `goal.intake.completion_proof` | The observable signal that closes the full original outcome. Recorded during intake; remains stable across the goal's lifetime. |

These three fields are related but distinct:

- `completion_proof` is the intake-level description of what done looks like.
- `oracle.signal` is the repeating live check applied during execution.
- `oracle.final_proof` is the closing evidence that maps the final audit receipt to reality.

### The oracle in `goal.md`

The charter's `## Goal Oracle` section holds the human-readable statement:

```markdown
## Goal Oracle

The oracle for this goal is:

`<specific observable signal>`

The PM must keep comparing task receipts to this oracle. Planning, discovery,
a passing tiny slice, or a clean-looking board is not enough. The goal finishes
only when a final Judge/PM audit maps receipts and verification back to this
oracle and records `full_outcome_complete: true`.
```

### Oracle types

The intake compiler records `proof_type` in `state.yaml`:

| Proof type | What it looks like |
|---|---|
| `test` | A test suite that passes, e.g. `npm test` exits 0 with specific test names listed |
| `demo` | A recorded or live walkthrough of the running application |
| `artifact` | A generated file, bundle, or deliverable that can be inspected |
| `metric` | A measurable benchmark: build time, error rate, latency |
| `review` | A human review sign-off for a PR or code change |
| `source_backed_answer` | An answer with citations to authoritative source material |
| `decision` | A formal go/no-go or architectural decision with rationale |

## Weak proof detection

The validator (`goalbuddy/scripts/check-goal-state.mjs` → `isWeakProof()`) detects placeholder oracle fields. A value is considered weak if it is:

- Empty or null
- The string `unknown`, `tbd`, `todo`, or `none`
- A template placeholder matching the pattern `<.*>` (e.g., `<live check goes here>`)

When `goal_pressure_requires_oracle: true` (the default), weak `oracle.signal` and `oracle.final_proof` produce warnings:

```json
{
  "warnings": [
    "goal.oracle.signal is missing or placeholder-like; weak oracles make /goal finish too early.",
    "goal.oracle.final_proof is missing or placeholder-like; final completion needs receipt-backed proof."
  ]
}
```

When `no_completion_on_weak_proof: true` (the default) and `goal.status` is `done`, these become hard errors.

A weak `intake.completion_proof` always produces a warning, regardless of goal status.

## The oracle at completion

A goal cannot close with `goal.status: done` until all of these are true:

1. `goal.intake.completion_proof` is concrete (not weak).
2. `goal.oracle.signal` is concrete (not weak).
3. `goal.oracle.final_proof` is concrete (not weak).
4. A final done Judge or PM audit task exists with `decision: complete`.
5. That same final audit receipt has `full_outcome_complete: true`.

The validator checks condition 5 separately: a receipt with `decision: complete` but `full_outcome_complete: false` does not satisfy it. Both must be present.

In `state.yaml`:

```yaml
    receipt:
      result: done
      decision: complete
      full_outcome_complete: true
      rationale: "All three billing routes verified; end-to-end test suite passes; demo transcript matches original request."
```

## Choosing a good oracle

A good oracle is a concrete observable that a skeptical person could verify independently. Examples by goal kind:

| Goal kind | Example oracle signal |
|---|---|
| Bug fix | `npm test -- test/auth/session.test.ts` exits 0; the specific reproduction step no longer triggers the error |
| Feature build | Browser walkthrough: user can complete the checkout flow end-to-end without errors |
| Refactor | All existing tests pass; `git diff --stat` shows no new files added |
| Documentation | Every internal link resolves; code examples produce the shown output |
| Research / audit | Source-backed answer with citations; no contradictory claims unresolved |

A bad oracle is a process milestone ("Judge has approved the plan"), a task count ("all 10 tasks done"), or a board state ("no blocked tasks").

## Interaction with other concepts

| Concept | Relationship |
|---|---|
| [The loop](./the-loop.md) | Step 4 of every loop turn re-checks `intake.completion_proof` and `likely_misfire`. Step 12 requires the final audit to map back to the oracle. |
| [The board](./the-board.md) | Oracle fields are stored in `state.yaml`; the `rules.goal_pressure_requires_oracle` and `rules.no_completion_on_weak_proof` flags control how the validator enforces them. |
| [Agents](./agents.md) | Judge is responsible for the final oracle mapping at completion, returning `full_outcome_complete: true` only when receipts and verification match the signal. |
| [Local board](./local-board.md) | The local board viewer shows `goal.tranche` and `goal.status` but does not surface oracle fields directly. Oracle pressure lives in the loop, not the visual display. |

## Common gotchas

**Template placeholders surviving into execution.** If `goal-prep` created a `state.yaml` with `signal: "<live check goes here>"` and the PM didn't update it, the validator will warn on every run and hard-error at completion. Fill in the oracle before starting Worker tasks.

**Conflating `completion_proof` and `oracle.signal`.** They serve different moments. `completion_proof` is the intake statement of what done looks like. `oracle.signal` is the live test that runs after each Worker package. Both must be concrete, and both are checked independently.

**`full_outcome_complete: true` on a partial result.** Judge must evaluate the full original user outcome, not just the current tranche or slice. A tranche that passes all its tasks but leaves the broader user outcome incomplete must return `full_outcome_complete: false` and queue the next safe Worker.

**Completing on process evidence.** "Judge approved the last slice" or "board shows all tasks done" are not oracle signals. The oracle is always an observable state of the world outside the board itself.
