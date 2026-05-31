# GoalBuddy 0.3.8: Board Hub Guardrails

Release date: 2026-05-29

This patch release fixes a confusing local-board failure mode.

GoalBuddy already supports multiple local boards on one shared hub:

```text
http://goalbuddy.localhost:41737/<slug>/
```

The problem was the unregistered-path error. If an agent opened a new board URL before registering that goal with the hub, the server returned a bare 404. That made it too easy to infer that the process on `41737` was stale, even when it was a healthy multi-board hub for another goal.

## What Changed

- Unregistered board paths now return an explicit diagnostic explaining that a `/slug/` 404 does not mean the hub is stale.
- The diagnostic points agents to `http://127.0.0.1:41737/api/boards` and tells them to rerun `npx goalbuddy board <goal-dir>` to register the goal on the same port.
- `$goal-prep` / `/goal-prep` now says to stop a process on `41737` only when `/api/boards` proves the listener is not a current GoalBuddy multi-board hub.
- `npm run check` now includes the local board surface tests and syntax checks.

## Release Boundaries

This release does not change the board state model. `state.yaml` remains the source of truth, the local board remains a viewer over repo files, and multiple boards still share the same local hub.

## Package Notes

This release updates:

- npm package version: `0.3.8`
- Codex plugin version: `0.3.8`
- Claude Code plugin version: `0.3.8`
- release checks to cover `goalbuddy/surfaces/local-goal-board/`

Before publishing, verify:

```bash
npm run check
npm run publish:check
```
