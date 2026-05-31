# Key concepts

Alphabetical glossary of GoalBuddy terms.

---

**Active task**
The single task currently being worked. GoalBuddy enforces exactly one active task at any time. No implementation work may happen outside an active Worker or PM task. The active task is recorded in `state.yaml` under `active_task`.

---

**Board**
The live kanban view of a single `state.yaml`. Each goal has its own board, accessible at `http://goalbuddy.localhost:41737/<slug>/`. The board is a read-only view; `state.yaml` is the authoritative source. Multiple boards share one local hub at port 41737, with an in-header board switcher.

---

**Charter**
The `goal.md` file. It records the user's original request, interpreted outcome, goal oracle, constraints, and current tranche. The charter is human-editable. When `goal.md` and `state.yaml` disagree on task status or completion, `state.yaml` wins.

---

**Goal kind**
The classification GoalBuddy assigns to a goal during intake. One of: `specific`, `open_ended`, `existing_plan`, `recovery`, `audit`. The kind determines the default first active task (e.g., vague `open_ended` goals start with Scout; goals with an `existing_plan` start with PM or Judge validation).

---

**Goal oracle**
The observable signal that proves the original owner outcome is actually true. May be a test suite, browser walkthrough, demo transcript, generated artifact, benchmark, source-backed answer, release check, or final human decision. The oracle is recorded in `state.yaml` under `goal.oracle.signal` and must be established before execution begins. Without an oracle there is no basis for completion — "no oracle, no serious goal."

---

**Judge**
A read-only agent role (`goal_judge.toml`) that handles phase gates, risky scope decisions, completion reviews, and ambiguity resolution. Judge uses high reasoning effort and runs in read-only sandbox mode. Judge's primary job is selecting the largest safe useful slice for the next Worker task and, at the end of a tranche, deciding whether `full_outcome_complete: true` can be recorded. Judge does not choose the next active task; the PM does that after receiving the Judge receipt.

---

**Micro-slice**
A task that is technically safe but outcome-light: adding one more helper, contract file, wrapper, projection, or doc note without moving the goal toward a real milestone. GoalBuddy warns when two consecutive micro-slices appear. The board checker (`check-goal-state.mjs`) and `goalbuddy prompt` emit non-fatal warnings. After two tiny tasks in a row, PM or Judge should reorient toward a vertical slice.

---

**Notes**
The `notes/` directory inside a goal folder. Used only when a Scout, Judge, or PM receipt is too large to fit inline on the task card in `state.yaml`. A note file is named `notes/<task-id>-<slug>.md` and referenced from the task receipt via `note: notes/T001-repo-map.md`.

---

**PM**
The main `/goal` thread. The PM owns board truth — it is the only role that may choose the active task, update `state.yaml`, mark tasks done, and declare the goal complete. Scout, Judge, and Worker return receipts; the PM decides what to do with them.

---

**Receipt**
A compact, durable record written to a task card in `state.yaml` when the task completes, blocks, or escalates. Scout receipts include evidence and findings. Worker receipts include changed files and command results. Judge receipts include decisions and `full_outcome_complete`. A receipt is the proof that a task happened and what it produced.

---

**Scout**
A read-only agent role (`goal_scout.toml`) that maps repos, gathers evidence, and returns findings. Scout uses low reasoning effort and runs in read-only sandbox mode. Scout does not plan, implement, or choose the next active task. Multiple Scouts may run in parallel because neither writes. See [`goalbuddy/agents/goal_scout.toml`](../goalbuddy/agents/goal_scout.toml).

---

**Slice sizing**
The policy for choosing how much work belongs in one Worker task: the largest safe useful slice. "Safe" means bounded, explicit, verified, and reversible — not small. A good Worker task completes a working screen, API path, data pipeline step, vertical slice, or real bug fix. Sizing is too small when tasks keep adding helpers or contracts without changing observable behavior.

---

**State (`state.yaml`)**
The machine truth for a goal. Tracks goal metadata, the oracle, intake details, rules, agent availability, visual board status, the active task, all task cards, and verification freshness. `state.yaml` overrides `goal.md` whenever they conflict. All receipts are written here.

---

**Subgoal**
A depth-1 child board that a parent task may link to for bounded branching work. Stored under `subgoals/` inside the goal directory, e.g., `subgoals/T003-child/state.yaml`. Subgoals are non-recursive (depth 1 only), linked from exactly one parent task, and rendered inside the parent task detail in the local board. A subgoal does not replace the parent board; `state.yaml` at the parent level remains authoritative.

---

**Tranche**
A bounded execution segment within a goal. A broad or open-ended goal should define a tranche rather than an unlimited mission — for example, "complete successive safe verified work packages until the add-OAuth scope is done." When a tranche finishes, a final audit confirms it and records what remains. Defining a tranche prevents forever-goals and gives the PM a concrete stopping condition.

---

**Worker**
A write-capable agent role (`goal_worker.toml`) that executes one coherent bounded work package. Worker uses medium reasoning effort and runs in workspace-write sandbox mode. Worker edits only files listed in `allowed_files`, runs the `verify` commands, and returns a receipt. At most one write-capable Worker may be active at any time. Worker does not decide whether its completion satisfies the full goal oracle — that is a Judge or PM decision.
