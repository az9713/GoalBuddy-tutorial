# What is GoalBuddy?

GoalBuddy is a structured operating loop that keeps Codex and Claude Code oriented during long multi-turn `/goal` runs by giving every goal a charter, a board, a goal oracle, and a proof requirement.

## The problem

Coding agents drift. In a long session, the model loses track of the original intent, skips verification, declares work done prematurely, or restarts the plan from scratch on every turn. The longer the run, the worse the drift.

The underlying cause is structural: without a durable work surface, each turn is a fresh context. The model re-interprets the goal, re-prioritizes work, and re-discovers what was already done. Receipts disappear into chat history. Blocked tasks look identical to queued tasks. There is no oracle keeping pressure on the original outcome.

GoalBuddy fixes this by writing the operating state into the repo rather than keeping it in the model's context window. The charter records intent. The board records task truth. Receipts record what happened. The oracle records what "done" actually means.

## The mental model

```text
Intent -> Oracle -> Surface -> Loop -> Proof
```

Every GoalBuddy run moves through five stages:

1. **Intent** — the user's original request, captured faithfully in `goal.md`.
2. **Oracle** — the observable signal that will prove the outcome is real (a test suite, demo transcript, benchmark, artifact, or final human decision). No oracle means no serious goal.
3. **Surface** — the local board at `http://goalbuddy.localhost:41737/<slug>/`, a live kanban view of `state.yaml`.
4. **Loop** — Scout maps, Judge selects the largest safe slice, Worker executes and leaves a receipt, PM advances the board and keeps the loop going.
5. **Proof** — a final Judge or PM audit maps receipts and verification back to the oracle and records `full_outcome_complete: true`.

## Architecture overview

```text
User intent
    |
    v
$goal-prep / /goal-prep
    |
    v
docs/goals/<slug>/
  goal.md           <-- charter: what the user wants
  state.yaml        <-- board truth: task state, receipts, oracle
  notes/            <-- long findings overflow
  .goalbuddy-board/ <-- generated local board files
  subgoals/         <-- optional depth-1 child boards
    |
    v
Local board: http://goalbuddy.localhost:41737/<slug>/
    |
    v
/goal Follow docs/goals/<slug>/goal.md.
    |
    +---> PM (main /goal thread)
    |       |
    |       +---> Scout (read-only mapper)
    |       +---> Judge (phase-gate reviewer)
    |       +---> Worker (bounded writer)
    |
    v
Receipt loop until: full_outcome_complete: true
```

`state.yaml` is always board truth. If `goal.md` and `state.yaml` disagree on task status or completion, `state.yaml` wins.

## How the pieces fit

A typical run looks like this.

`$goal-prep` (Codex) or `/goal-prep` (Claude Code) runs the intake compiler, asks diagnostic questions for vague goals, creates `goal.md` and `state.yaml`, starts the local board, and prints the exact command to run next.

The user runs `/goal Follow docs/goals/<slug>/goal.md.` The PM reads the charter, reads the board, and activates the first task. For a vague goal, the first task is usually Scout.

Scout runs read-only, inspects the repo, and returns a compact evidence receipt — candidate improvements, verification commands, high-leverage files. Scout does not choose the next task.

Judge reads the Scout receipt and chooses the largest safe useful slice: the biggest chunk of work that is bounded, explicit, verified, and reversible. Judge returns a decision receipt naming the Worker objective, `allowed_files`, `verify` commands, and `stop_if` conditions.

Worker executes the whole assigned slice, runs the verify commands, and returns a receipt with changed files, command results, and a summary. Worker edits only files named in `allowed_files`.

The PM writes the receipt to `state.yaml`, chooses the next active task, and keeps the loop going. The goal ends only when a final Judge or PM audit maps all receipts back to the oracle and records `full_outcome_complete: true`.

## What this is not

| This is not | Why |
|---|---|
| A replacement for native Codex `/goal` | GoalBuddy prepares boards and prompts for the native `/goal` feature; it does not enable or replace it |
| A plugin marketplace | The local board is a built-in view of `state.yaml`; custom integrations are ordinary repo tasks |
| A parallel execution engine | `goalbuddy parallel-plan` is read-only inspection; GoalBuddy does not spawn or coordinate parallel Workers unless write scopes are provably disjoint |
| Right for single-file changes | Creating a GoalBuddy board for a one-change task adds overhead without benefit |
| A planning-only tool | Planning, discovery, and task selection are not terminal outcomes; execution goals require implementation evidence before completion |
