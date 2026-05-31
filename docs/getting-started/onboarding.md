# Onboarding

## If you've used GitHub Projects or Trello, GoalBuddy is like...

...a project board that lives inside your repo, is written and read by your AI coding agent, enforces one active task at a time, and requires verifiable proof before it will close a card.

The key difference: GoalBuddy's board is not for humans to manage manually. It is the agent's own state file. The agent reads it at the start of every turn, writes a receipt when a task finishes, picks the next task, and keeps going. You watch the board; the agent runs it.

## Building the mental model

Four concepts do all the work.

**Charter.** The `goal.md` file records what you asked for, why it matters, what "done" looks like, and what is explicitly out of scope. It is the editable human layer. Once execution starts, `state.yaml` takes over as the source of truth — but the charter is where intent lives.

**Board.** The `state.yaml` file is the machine-readable task list. Every task card has an id, a role (Scout / Judge / Worker / PM), a status, an objective, constraints, expected output, and a receipt field. The local board at `http://goalbuddy.localhost:41737/<slug>/` is just a live view of this file.

**Oracle.** Before any task runs, GoalBuddy requires an observable signal that will prove the original outcome is real. This might be a test suite that goes green, a browser walkthrough recording, a generated artifact, or a benchmark number. The oracle is recorded in `state.yaml` and the PM compares every receipt back to it. Planning is not enough. Implementation without passing the oracle is not enough. Only when a final Judge or PM audit maps receipts to the oracle can `full_outcome_complete: true` be recorded.

**Loop.** The execution loop is: Scout maps the evidence, Judge selects the largest safe slice, Worker executes and leaves a receipt, PM advances the board and picks the next task. This repeats until the oracle passes and the final audit closes the goal.

```text
Intent -> Oracle -> Surface -> Loop -> Proof
```

## A realistic scenario: "Add OAuth to the API"

You open Claude Code and run:

```text
/goal-prep Add OAuth login to our Express API.
```

Goal Prep asks one diagnostic question: what would convince you OAuth is working? You answer: the existing test suite passes, plus a manual walkthrough where a GitHub login redirects back and a JWT is set in the session.

Goal Prep records that as the oracle, classifies the goal as `specific`, creates `docs/goals/add-oauth-api/`, starts the local board, and prints:

```text
/goal Follow docs/goals/add-oauth-api/goal.md.
```

You open the board URL in your browser. One task card appears: T001, Scout, active.

You run the `/goal` command.

**Turn 1 — Scout.** Scout reads `package.json`, the existing auth middleware, and the test suite. It does not touch implementation files. It returns a receipt: passport.js is already a dependency but unused; three test files cover the current session logic and will break if middleware is inserted naively; the `allowed_files` for a safe first Worker slice are `src/auth/oauth.js`, `src/routes/auth.js`, and `test/auth/oauth.test.js`.

**Turn 2 — Judge.** Judge reads the Scout receipt, reviews the three test files, and decides the largest safe slice is: implement the GitHub OAuth callback handler, wire it into the existing auth router, and make the three test files pass. Judge writes a receipt naming the exact `allowed_files`, the `verify` command (`npm test -- test/auth/`), and two `stop_if` conditions.

**Turn 3 — Worker.** Worker implements the callback handler and router wiring inside the named files only. It runs `npm test -- test/auth/` twice — first run has two failures; Worker fixes them; second run passes. Worker writes a receipt with the changed files, command results, and a 90-word summary.

**Turn 4 — PM.** The PM reads the Worker receipt, checks it against the oracle (test suite passes — one of two required signals), and decides the next task is a PM-level board task: write a walkthrough prompt for the manual session check. That is the second oracle signal.

**Turn 5 — final audit.** Judge reads all four prior receipts, confirms the test suite is green, confirms the walkthrough script covers the GitHub-redirects-to-JWT path, and records `full_outcome_complete: true`.

The goal closes. The board shows every receipt. Nothing is in chat history; everything is in `docs/goals/add-oauth-api/state.yaml`.

## Why it works this way: three surprising design choices

**1. `state.yaml` beats `goal.md` when they conflict.**

Most users expect the charter to be authoritative — it is the human-written document. GoalBuddy inverts this: the charter is editable and readable, but `state.yaml` is the machine truth for task status, receipts, the active task, and completion. The reason is that the charter can become stale between turns; `state.yaml` is always written by the agent immediately after each action.

**2. The oracle is required before the first task runs.**

It is tempting to start implementation and figure out what "done" means later. GoalBuddy refuses: if there is no oracle, there is no gate for completion, and the PM will eventually mark the goal done based on effort rather than outcome. Recording the oracle first keeps every receipt accountable to an external signal the user actually cares about.

**3. Safe does not mean small.**

GoalBuddy actively warns when tasks are too small. A task that adds one helper function, one contract file, or one projection without changing observable behavior is a micro-slice. After two micro-slices in a row, the board checker emits a warning and Judge is expected to select a vertical slice instead. The goal is the largest safe useful slice — a working screen, a working API path, a real bug fix — not the smallest safe task.

## Where to go next

- [What is GoalBuddy?](../overview/what-is-goalbuddy.md) — full architecture and the complete mental model
- [Key concepts](../overview/key-concepts.md) — definitions for every term used above
- [Quickstart](quickstart.md) — install and run your first goal
- [The loop](../concepts/the-loop.md) — deep dive into the PM execution loop
