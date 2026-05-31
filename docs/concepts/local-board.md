# The local board

The local board is a live-updating browser viewer for a GoalBuddy goal. It reads `state.yaml`, renders a four-column Kanban layout, and pushes changes in real time via Server-Sent Events whenever `state.yaml` or `notes/` changes.

## Why it exists as its own concept

A goal board that lives only in YAML is hard to scan during a long autonomous run. The local board gives you and the AI coding agent a live view of task cards moving between columns, receipts appearing, and the active task indicator updating — without requiring any external service, credentials, or cloud integration.

## How it works

The server is implemented in `goalbuddy/surfaces/local-goal-board/scripts/local-goal-board.mjs`. It is started with:

```bash
npx goalbuddy board docs/goals/<slug>
```

Expected output:

```text
GoalBuddy local board: http://goalbuddy.localhost:41737/my-goal/
GoalBuddy local hub: http://goalbuddy.localhost:41737/
Watching: docs/goals/my-goal/state.yaml
Press Ctrl-C to stop.
```

### Multi-board hub

The first `npx goalbuddy board` call that succeeds on port 41737 becomes the hub. All subsequent calls for different goals POST to `/api/boards` and register with the existing hub instead of starting a new server:

```text
GoalBuddy local board: http://goalbuddy.localhost:41737/other-goal/
GoalBuddy local hub: http://goalbuddy.localhost:41737/
Registered with the existing GoalBuddy local board hub.
```

If port 41737 is in use by something that is not a GoalBuddy hub (i.e., `POST /api/boards` returns 404), the server prints an actionable error:

```text
Port 41737 is already in use, but it is not the GoalBuddy multi-board hub.
Stop the existing local board process on 127.0.0.1:41737, then retry.
```

> **Note:** If `http://goalbuddy.localhost:41737/<slug>/` returns 404, do not stop the existing hub. Check `http://127.0.0.1:41737/api/boards` first. If that returns GoalBuddy board JSON, the hub is healthy — rerun `npx goalbuddy board docs/goals/<slug>` to register the new goal on the same port. Source: `local-goal-board.mjs` → `registerWithBoardHub()`.

### File watching

The server watches `state.yaml` and the `notes/` directory using Node.js `fs.watch`. Changes are debounced at 80 ms before a board reload is triggered (`local-goal-board.mjs` → `debounce(onChange, 80)`). This means rapid sequential writes (PM writes a receipt then immediately updates the active task pointer) arrive as a single update to the browser.

If the board's `state.yaml` links to subgoal boards via `subgoal.path`, the watcher automatically expands to include those child goal directories. When a child board's `state.yaml` changes, the parent board's SSE stream rebroadcasts the full payload.

### Generated files

On first run (and on each `--once` invocation), the server writes four files into the goal directory (`lib/goal-board.mjs` → `writeBoardApp()`):

```text
docs/goals/<slug>/
  .goalbuddy-board/
    index.html
    styles.css
    app.js
    goalbuddy-mark.png
```

These files are self-contained. `app.js` connects to `./api/board` and `./events` relative to the board URL.

## CLI reference

| Flag | Default | Description |
|---|---|---|
| `--goal <path>` | (required) | Goal directory containing `state.yaml`. Positional argument also accepted. |
| `--host <host>` | `127.0.0.1` | Bind host. Also sets the public hostname for printed URLs. |
| `--port <port>` | `41737` | Server port. |
| `--once` | off | Generate `.goalbuddy-board/` and exit without starting a server. |
| `--json` | off | Print structured JSON output instead of human-readable lines. |

The default bind host is `127.0.0.1` and the default public hostname in printed URLs is `goalbuddy.localhost`. The server listens only on loopback; the friendly hostname makes printed URLs clickable in the agent's in-app browser.

## API endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/boards` | List all registered boards |
| `POST` | `/api/boards` | Register a new board. Body: `{ "goalDir": "<path>" }` |
| `GET` | `/api/settings` | Read current viewer settings |
| `PUT` | `/api/settings` | Save viewer settings. Body: `{ "settings": { ... } }` |
| `GET` | `/<slug>/api/board` | Current board payload for the named goal |
| `GET` | `/<slug>/events` | SSE stream of board updates (event: `board`, data: JSON payload) |
| `GET` | `/` or `/boards` | Redirect to the preferred board (last viewed or newest) |
| `GET` | `/<slug>/` | Serve the static board viewer app |

If `state.yaml` cannot be parsed, `/<slug>/api/board` returns a payload with an `error` field and the viewer displays a parse-error card rather than crashing.

## Viewer settings

Settings persist to `~/.goalbuddy/local-board-settings.json` on the server and are mirrored in `localStorage` in the browser. Override the path with `GOALBUDDY_LOCAL_BOARD_SETTINGS_PATH`.

| Setting key | Options | Default | Description |
|---|---|---|---|
| `theme` | `system`, `light`, `dark` | `system` | Color scheme. `system` follows OS preference. |
| `density` | `comfortable`, `compact` | `comfortable` | Card spacing and padding. |
| `completedVisibility` | `show`, `collapse` | `show` | Whether the Completed column shows cards or collapses. |
| `boardOpenBehavior` | `last`, `newest` | `last` | Which board to redirect to when opening the hub root. |
| `motion` | `system`, `reduce`, `allow` | `system` | Card animation behavior. `reduce` disables transitions and the active-card orbit. |

## The board columns

| Column | Task statuses shown | Description |
|---|---|---|
| Todo | `queued` | Queued work ready to pull |
| In Progress | `active` | The active task (at most one) |
| Blocked | `blocked` | Needs unblock or a smaller slice |
| Completed | `done` | Receipted work |

Within the Completed column, cards are sorted newest-first. Within other columns, cards are sorted by id.

## Active task card

The active task card renders with a distinguishing animated border gradient (blue/indigo/teal in light mode; deep blues/purples in dark mode). When `motion: reduce` is set or the OS `prefers-reduced-motion` media query matches, the animation is replaced with a static low-opacity highlight.

## Subgoal rendering

When a task card has a linked subgoal (`subgoal.path` in `state.yaml`), the card's detail panel (opened by clicking the card) shows a miniature four-column board for the child goal. Child task cards show id, status, assignee badge, and receipt presence. The board switcher in the top navigation bar shows all registered boards, labeling child goals as `Child: <title>`.

## Interaction with other concepts

| Concept | Relationship |
|---|---|
| [The board](./the-board.md) | The local board server reads `state.yaml`. The board concept defines what valid content looks like. |
| [The loop](./the-loop.md) | Every loop turn that writes a receipt or updates the active task pointer triggers the 80ms debounce, which pushes an update to any open viewer. |
| [Agents](./agents.md) | Agent role badges appear on each card. The viewer reflects agent availability from the `agents` block but does not enforce it. |
| [The oracle](./oracle.md) | The viewer shows `goal.tranche` and `goal.status` but does not display oracle fields directly. Oracle pressure is a loop-level concern. |

## Common gotchas

**Killing the hub to fix a 404.** A `/<slug>/` returning 404 means the slug has not been registered with the hub yet — not that the hub is broken. Rerun `npx goalbuddy board docs/goals/<slug>` to register, then use the printed URL.

**Port conflict with a non-GoalBuddy process.** If something else is on 41737, the fallback POST to `/api/boards` will return 404 and the server will print an error. You must stop the conflicting process.

**Large `state.yaml` parse errors.** The board uses a lightweight custom YAML parser in `lib/goal-board.mjs` that does not support block scalars (`|` or `>`). Receipt summaries written with block scalar syntax will cause a parse error visible in the viewer. Use quoted or plain scalar strings in `state.yaml`.

**Settings not persisting between machines.** Settings are stored in `~/.goalbuddy/local-board-settings.json` on the local filesystem. They are not synced; each machine starts from defaults.
