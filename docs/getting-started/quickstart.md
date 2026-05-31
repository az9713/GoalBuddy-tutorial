# Quickstart

Time to first `/goal` run: approximately 3 minutes.

## Steps

### 1. Install GoalBuddy

```bash
npx goalbuddy
```

This installs the GoalBuddy skill and Scout/Judge/Worker agent configs into both Codex and Claude Code. You will see confirmation output from the installer.

### 2. Restart your agent

Restart Codex or Claude Code so the new skill and agents are loaded.

### 3. Run goal prep

In Codex:

```text
$goal-prep
```

In Claude Code:

```text
/goal-prep
```

Goal Prep runs the intake compiler, asks one or more diagnostic questions if your request is vague, creates the goal directory, starts the local board, and prints the exact command to run next.

Expected output (after you provide your goal):

```text
/goal Follow docs/goals/<your-goal-slug>/goal.md.
```

Goal Prep also prints a clickable board link:

```text
Open GoalBuddy board: http://goalbuddy.localhost:41737/<your-goal-slug>/
```

### 4. Open the local board

If the board did not open automatically, start it from your terminal:

```bash
npx goalbuddy board docs/goals/<your-goal-slug>
```

Expected output:

```text
GoalBuddy local board: http://goalbuddy.localhost:41737/<your-goal-slug>/
GoalBuddy local hub: http://goalbuddy.localhost:41737/
Watching: docs/goals/<your-goal-slug>/state.yaml
Press Ctrl-C to stop.
```

Open the board URL in your browser or the agent's in-app preview to watch task cards populate in real time.

### 5. Run `/goal`

Copy the command Goal Prep printed and run it in your agent:

```text
/goal Follow docs/goals/<your-goal-slug>/goal.md.
```

The PM reads the charter and board, activates the first task, and begins the Scout → Judge → Worker → receipt loop.

## What just happened

`npx goalbuddy` installed the skill and agents. Goal Prep compiled your intent into a charter (`goal.md`), a seeded board (`state.yaml`), and a `notes/` directory under `docs/goals/<slug>/`. The local board server makes `state.yaml` readable as a live kanban. The `/goal` command hands control to the PM, which runs Scout to map the repo, Judge to select the first safe work slice, and Worker to execute it — writing receipts back to `state.yaml` after each task and continuing until a final audit records `full_outcome_complete: true`.

## Next steps

- [Onboarding guide](onboarding.md) — understand the mental model with a realistic worked example
- [Key concepts](../overview/key-concepts.md) — definitions for every GoalBuddy term
- [What is GoalBuddy?](../overview/what-is-goalbuddy.md) — deeper architecture overview
