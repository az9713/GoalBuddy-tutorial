# System design

GoalBuddy is an npm package that installs as a skill and plugin into Codex and Claude Code. Its architecture has six components — skill payload, templates, agents, local board, scripts, and CLI — all interacting through two durable files: `goal.md` (intent) and `state.yaml` (truth).

---

## Architecture diagram

```text
                     User
                       |
                  goal-prep / /goal-prep
                  (SKILL.md system prompt)
                       |
                  Intake Compiler
                       |
           +-----------+-----------+
           |                       |
        goal.md               state.yaml
        (charter)             (board truth)
           |                       |
           +----------+------------+
                      |
               /goal Follow ...
                      |
                   PM Thread
                   (main /goal thread)
                   |        |        |
                Scout    Worker    Judge
              (read-only) (bounded  (read-only
                          write)     gate)
                      |
              Receipt written to
              state.yaml
                      |
              local board server
              (SSE -> browser)
              http://goalbuddy.localhost:41737/<slug>/
```

---

## Component breakdown

### 1. Skill payload (`goalbuddy/SKILL.md`)

The system prompt installed into Codex as `$goal-prep` and into Claude Code as `/goal-prep`. This is the largest component — it defines the full operating behavior: intake compiler, diagnostic ladder, board creation, the PM loop, slice sizing policy, and agent dispatch rules.

The skill also defines the `/goal Follow ...` command's behavior: the PM reads `goal.md` and `state.yaml` on every turn and follows the 12-step loop.

**Install location:**
- Codex: `~/.codex/plugins/cache/goalbuddy/goalbuddy/<version>/SKILL.md`
- Claude Code: `~/.claude/plugins/goalbuddy/skills/goalbuddy/SKILL.md`

### 2. Templates (`goalbuddy/templates/`)

Three template files used by `goal-prep` to create new goal directories:

| Template | Creates | Purpose |
|---|---|---|
| `goal.md` | `docs/goals/<slug>/goal.md` | Human-editable charter |
| `state.yaml` | `docs/goals/<slug>/state.yaml` | Machine truth board |
| `note.md` | `docs/goals/<slug>/notes/*.md` | Overflow findings for long receipts |

### 3. Agents (`goalbuddy/agents/`)

TOML agent configs installed to `~/.codex/agents/`. Each defines a role, reasoning effort, sandbox mode, and the full system prompt for the agent.

| File | Agent | Reasoning | Sandbox | Role |
|---|---|---|---|---|
| `goal_scout.toml` | Scout | `low` | `read-only` | Read-only repo mapper |
| `goal_worker.toml` | Worker | `medium` | `workspace-write` | Bounded writer for one slice |
| `goal_judge.toml` | Judge | `high` | `read-only` | Skeptical phase-gate reviewer |

Claude Code markdown agent equivalents live in `plugins/goalbuddy/agents/`.

All three agents return a structured JSON receipt (`goalbuddy_receipt_v1`). The PM validates and writes the receipt to `state.yaml`.

### 4. Local board (`goalbuddy/surfaces/local-goal-board/`)

A Node.js HTTP + SSE server that serves a live Kanban view of `state.yaml`.

**Key files:**
- `scripts/local-goal-board.mjs` — HTTP server, multi-board hub, file watcher, SSE broadcaster
- `scripts/lib/goal-board.mjs` — YAML parser, board payload builder, board HTML/CSS/JS generator

**Data flow:**
1. `npx goalbuddy board docs/goals/<slug>` starts the server on port 41737
2. The server writes `.goalbuddy-board/` (index.html, styles.css, app.js) into the goal directory
3. `fs.watch` monitors `state.yaml` and `notes/` with an 80ms debounce
4. On change: `createBoardPayload()` parses `state.yaml`, builds the board JSON
5. The JSON is pushed to all connected browser clients via SSE (`event: board`)
6. `app.js` in the browser receives SSE events and re-renders the board using FLIP animations

**Multi-board hub pattern:**
- First server call on port 41737 → becomes the hub
- Subsequent calls → POST `/api/boards` to register with the hub
- Hub stores all boards in a `Map`; board switcher rendered in the browser

### 5. Scripts (`goalbuddy/scripts/`)

Three utility scripts called during goal runs:

| Script | Invoked via | Purpose |
|---|---|---|
| `check-goal-state.mjs` | `node <path> docs/goals/<slug>/state.yaml` | Validates `state.yaml` against all v2 rules |
| `render-task-prompt.mjs` | `npx goalbuddy prompt <goal-dir>` | Renders compact dispatch prompt for active task |
| `parallel-plan.mjs` | `npx goalbuddy parallel-plan <goal-dir>` | Analyzes parallel safety across all boards |
| `check-update.mjs` | Called by `goal-prep` at turn start | Checks npm for a newer GoalBuddy version |

`render-task-prompt.mjs` and `parallel-plan.mjs` both import from `lib/goal-board.mjs` for YAML parsing.

### 6. CLI (`internal/cli/`)

The `npx goalbuddy` entry point and installer. Not part of the installable skill — lives in `internal/` which is package-only infrastructure.

| File | Purpose |
|---|---|
| `goal-maker.mjs` | Main CLI entry point; routes subcommands |
| `postinstall.mjs` | Runs after `npm install` to set up skill and agents |
| `check-publish-version.mjs` | Pre-publish check used by `npm run prepublishOnly` |

---

## Data flows

### Goal creation flow

```text
User: "$goal-prep Build an OAuth login"
  -> SKILL.md (goal-prep invocation)
  -> Intake Compiler (private, classifies intent)
  -> Diagnostic ladder (if vague)
  -> Creates: docs/goals/<slug>/goal.md
              docs/goals/<slug>/state.yaml  (seeded task board)
              docs/goals/<slug>/notes/
  -> Runs: npx goalbuddy board docs/goals/<slug>
  -> Prints: /goal Follow docs/goals/<slug>/goal.md.
```

### Execution loop flow

```text
User: "/goal Follow docs/goals/<slug>/goal.md."
  -> PM reads goal.md (charter)
  -> PM reads state.yaml (finds active_task: T001)
  -> PM runs check-goal-state.mjs (validates board)
  -> PM activates T001 (Scout task)
  -> Scout: reads repo, returns JSON receipt
  -> PM: writes receipt to state.yaml -> T001 status: done
  -> state.yaml change -> local-goal-board.mjs watcher fires
  -> SSE event pushed to browser -> board rerenders
  -> PM: sets T002 active (Judge task)
  -> ... (loop continues)
  -> Final Judge audit: full_outcome_complete: true
  -> PM: sets goal.status: done
```

### Agent dispatch flow (with Codex subagents)

```text
PM: npx goalbuddy prompt docs/goals/<slug>
  -> render-task-prompt.mjs reads state.yaml
  -> Returns: metadata.required_spawn_agent_type = "goal_worker"
              task fields (allowed_files, verify, stop_if)
              receipt_schema
PM: spawn_agent(agent_type: "goal_worker", prompt: <rendered prompt>)
  -> goal_worker agent executes bounded Worker task
  -> Returns: goalbuddy_receipt_v1 JSON
PM: writes receipt to state.yaml
PM: advances board to next task
```

### Parallel safety analysis flow

```text
npx goalbuddy parallel-plan docs/goals/<slug>
  -> parallel-plan.mjs reads state.yaml
  -> Loads child board state.yamls (via subgoal.path links)
  -> For each active task:
       Scout/Judge: safe_to_parallelize: true (read-only)
       Worker: compare allowed_files across all Workers
               disjoint -> safe_to_parallelize: true
               overlapping or unknown -> false
  -> Prints recommendations (does NOT mutate state)
```

---

## Key design decisions

### 1. `state.yaml` as the single source of truth

**Decision:** Machine truth lives in `state.yaml`, not in `goal.md` or the model's context. When the two disagree, `state.yaml` wins.

**Why:** Long coding sessions suffer from context drift — the model re-interprets goals, forgets receipts, or re-plans from scratch. Keeping truth in a file that persists across turns makes the system resumable, auditable, and independent of context window size. A human can open `state.yaml` and see exactly what the agent knows.

### 2. Oracle required before execution

**Decision:** The goal oracle signal must be a concrete observable before Scout runs. Placeholder values (`<...>`, `tbd`, `unknown`) are detected and warned against; at completion they become hard errors.

**Why:** Without a forcing function, the PM has no external check to map receipts to. It will eventually mark the goal done based on effort and task completion rather than user outcome. The oracle keeps completion pressure grounded in reality. Source: `check-goal-state.mjs` → `isWeakProof()`.

### 3. One active task at a time (for write-capable tasks)

**Decision:** At most one task with `status: active` is allowed at any time. For Workers, at most one write-capable Worker may run simultaneously unless `parallel-plan` proves disjoint write scopes.

**Why:** Multiple concurrent Workers writing to overlapping files would produce undefined state and corrupt `state.yaml`. Scout tasks are exempt because they are read-only. The one-active-task rule makes the board state deterministic and the validator simple.

### 4. PM owns board truth

**Decision:** Scout, Worker, and Judge return receipts in JSON. The PM (main `/goal` thread) writes receipts to `state.yaml`, chooses the next active task, and marks completion. Agents never update the board directly.

**Why:** Giving agents write access to the board would allow them to choose the next task (violating the PM's orchestration role), create phantom task completions, or close the goal prematurely. The PM is the only role with full context across all receipts and the oracle.

### 5. Lightweight custom YAML parser

**Decision:** The local board and scripts use a hand-rolled YAML parser (`lib/goal-board.mjs` → `parseGoalStateText()`) rather than a YAML library.

**Why:** GoalBuddy's runtime has zero npm dependencies. The custom parser handles the specific YAML subset used in `state.yaml` (scalars, sequences, mappings, inline arrays). Block scalars (`|`, `>`) are not supported — receipts and summaries must use quoted or plain scalar strings.

---

## External dependencies

GoalBuddy has no runtime npm dependencies. All functionality uses Node.js built-ins:

| Built-in | Used for |
|---|---|
| `node:http` | Local board HTTP server |
| `node:fs` / `node:fs/promises` | File reading, writing, `fs.watch` |
| `node:path` | Path manipulation |
| `node:os` | `homedir()` for settings file path |
| `node:child_process` | `spawnSync` for child state validation |
| `node:url` | `fileURLToPath`, `URL` |

The browser client (`app.js`) uses vanilla ES modules and the `EventSource` and `fetch` APIs — no bundler required.
