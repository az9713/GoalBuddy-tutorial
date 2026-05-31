# goal.md reference

`goal.md` is the human-editable charter for a GoalBuddy goal. It is created by `goal-prep` from the `goalbuddy/templates/goal.md` template and lives at `docs/goals/<slug>/goal.md`.

> **Note:** When `goal.md` and `state.yaml` disagree on task status, active task, or completion, `state.yaml` wins. The charter is the intent layer; the board is machine truth.

---

## Section: `# <Goal Title>` (H1)

The goal title. Should match `goal.title` in `state.yaml`.

---

## Section: `## Objective`

A user-editable description of what the current tranche is trying to accomplish. Should be bounded to the current run, not an infinite mission.

**Example:**
```markdown
## Objective

Implement the GitHub OAuth callback handler, wire it into the auth router,
and make the three affected auth test files pass.
```

---

## Section: `## Original Request`

The shortest faithful copy of what the user asked for. If the user provided a plan, preserve it here or summarize it under Intake Summary.

**Example:**
```markdown
## Original Request

Add OAuth login to our Express API.
```

---

## Section: `## Intake Summary`

Structured output from the intake compiler. Populated by `goal-prep`; do not edit manually unless correcting an intake error.

| Field | Values | Description |
|---|---|---|
| `Input shape` | `vague`, `specific`, `existing_plan`, `recovery`, `audit` | How the request was classified |
| `Audience` | Any | Who benefits from the outcome |
| `Authority` | `requested`, `approved`, `inferred`, `needs_approval`, `blocked` | Whether the goal has authorization to proceed |
| `Proof type` | `test`, `demo`, `artifact`, `metric`, `review`, `source_backed_answer`, `decision` | What kind of evidence closes the goal |
| `Completion proof` | Any | Observable signal that closes the full original outcome |
| `Goal oracle` | Any | Live check that keeps pressure on the goal during execution |
| `Likely misfire` | Any | How `/goal` could succeed at the wrong thing |
| `Blind spots considered` | Any | Risks or unstated choices surfaced during diagnostic intake |
| `Existing plan facts` | Any | User-provided steps, files, constraints to preserve and validate |

**Example:**
```markdown
## Intake Summary

- Input shape: `specific`
- Audience: API users
- Authority: `approved`
- Proof type: `test`
- Completion proof: `npm test -- test/auth/ exits 0; manual walkthrough shows JWT in session`
- Goal oracle: `npm test -- test/auth/` run after each Worker package
- Likely misfire: Implementing OAuth UI rather than backend callback handler
- Blind spots considered: Session storage compatibility with existing middleware
- Existing plan facts: none
```

---

## Section: `## Goal Oracle`

Repeats the oracle from Intake Summary in the charter's narrative format. The PM re-reads this on every loop turn.

**Example:**
```markdown
## Goal Oracle

The oracle for this goal is:

`npm test -- test/auth/ exits 0 and all assertions pass`

The PM must keep comparing task receipts to this oracle. Planning, discovery,
a passing tiny slice, or a clean-looking board is not enough. The goal finishes
only when a final Judge/PM audit maps receipts and verification back to this
oracle and records `full_outcome_complete: true`.
```

---

## Section: `## Goal Kind`

One of: `specific`, `open_ended`, `existing_plan`, `recovery`, `audit`.

Must match `goal.kind` in `state.yaml`.

---

## Section: `## Current Tranche`

Describes the scope and stopping criterion for the current execution run.

**Default for continuous execution goals:**
```markdown
## Current Tranche

Discover enough evidence, choose the largest reversible local work package,
implement it, verify it, review only at phase/risk/final boundaries, then
immediately advance to the next work package until the full original outcome
is complete.
```

---

## Section: `## Non-Negotiable Constraints`

A list of hard constraints the PM and agents must respect regardless of other decisions.

**Example:**
```markdown
## Non-Negotiable Constraints

- Do not modify `src/auth/session.ts` — it is owned by the platform team.
- All verify commands must include `npm test -- test/auth/`.
- Do not add new npm dependencies without approval.
```

---

## Section: `## Stop Rule`

Documents when the goal may stop. The default stop rule from the template:

```markdown
## Stop Rule

Stop only when a final audit proves the full original outcome is complete.

Do not stop after planning, discovery, or Judge selection if the user asked
for working software and a safe Worker task can be activated.

Do not stop after a single verified Worker package when the broader outcome
still has safe local follow-up work.
```

---

## Section: `## Slice Sizing`

Documents the slice sizing policy. The default from the template:

```markdown
## Slice Sizing

Safe means bounded, explicit, verified, and reversible. It does not mean tiny.

A good task is the largest safe useful slice.
```

---

## Section: `## Canonical Board`

Points to `state.yaml` as machine truth.

```markdown
## Canonical Board

Machine truth lives at:

`docs/goals/<slug>/state.yaml`

If this charter and `state.yaml` disagree, `state.yaml` wins for task status,
active task, receipts, verification freshness, and completion truth.
```

---

## Section: `## Run Command`

The exact command to continue the goal. Must match the slug.

```markdown
## Run Command

```text
/goal Follow docs/goals/<slug>/goal.md.
```
```

---

## Section: `## PM Loop`

The 12-step loop the PM follows on every `/goal` continuation. This section is read by the PM on every turn; do not remove it.

```markdown
## PM Loop

On every `/goal` continuation:

1. Read this charter.
2. Read `state.yaml`.
3. Run the bundled GoalBuddy update checker when available and mention a newer version without blocking.
4. Re-check the intake: original request, input shape, authority, proof, blind spots, existing plan facts, and likely misfire.
5. Work only on the active board task.
6. Assign Scout, Judge, Worker, or PM according to the task.
7. Write a compact task receipt.
8. Update the board.
9. If safe local work remains, choose the next largest reversible Worker package and continue unless blocked.
10. If a problem, suggestion, or follow-up should become a repo artifact, create an approved issue/PR or ask the operator whether to create one.
11. Review at phase, risk, rejected-verification, ambiguity, or final-completion boundaries; do not review every small Worker by habit.
12. Finish only with a Judge/PM audit receipt that maps receipts and verification back to the original user outcome and records `full_outcome_complete: true`.
```
