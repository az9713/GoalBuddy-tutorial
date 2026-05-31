# Common issues

For task-specific troubleshooting, see the individual guides:
- [Prepare a goal](../guides/prepare-a-goal.md)
- [Run a goal](../guides/run-a-goal.md)
- [Write a receipt](../guides/write-a-receipt.md)
- [Use parallel agents](../guides/use-parallel-agents.md)

---

## Board URL returns 404

**Cause:** The GoalBuddy multi-board hub is running on port 41737, but the goal is not registered with it. A 404 on a slug path (`/my-goal/`) does not mean the hub is stale.

**Fix:**

1. Confirm the hub is a GoalBuddy hub:

```bash
curl http://127.0.0.1:41737/api/boards
```

2. If that returns GoalBuddy board JSON, register the goal on the existing hub:

```bash
npx goalbuddy board docs/goals/<slug>
```

3. If `/api/boards` returns 404 or non-GoalBuddy content, a different process owns port 41737. Identify and stop that process, then rerun `npx goalbuddy board docs/goals/<slug>`.

**If that doesn't work:** Check what is bound to 41737. On macOS/Linux: `lsof -i :41737`. On Windows: `netstat -ano | findstr :41737`. Stop the conflicting process, then retry.

---

## Port 41737 is in use but not by GoalBuddy

**Cause:** A different process is listening on port 41737. GoalBuddy cannot start its hub on the default port.

**Fix:**

1. Identify the owner:

```bash
curl http://127.0.0.1:41737/api/boards
```

If the response has no `boards` array, the port belongs to something else.

2. Stop the conflicting process.

3. Start the GoalBuddy hub:

```bash
npx goalbuddy board docs/goals/<slug>
```

---

## Agents show as `bundled_not_installed`

**Cause:** The GoalBuddy TOML templates for `goal_scout`, `goal_worker`, and `goal_judge` exist inside the package but are not installed to `~/.codex/agents/`.

**Fix:**

```bash
npx goalbuddy agents
```

If the templates are also missing:

```bash
npx goalbuddy install
```

**If that doesn't work:** Verify the install location manually. Agent configs should appear at:

```text
~/.codex/agents/goal_judge.toml
~/.codex/agents/goal_scout.toml
~/.codex/agents/goal_worker.toml
```

If absent after `npx goalbuddy install`, reinstall GoalBuddy fully: `npx goalbuddy`. Then run `npx goalbuddy doctor --target codex --goal-ready`.

> **Note:** `bundled_not_installed` is a warning, not a failure. `/goal` continues through PM fallback without installed agent configs.

---

## `goal.status` is stuck as `blocked`

**Cause:** The entire goal is marked `blocked` in `state.yaml` when only a specific task should be blocked. Continuous goals must keep `goal.status: active`.

**Fix:**

1. Open `docs/goals/<slug>/state.yaml`.
2. Set `goal.status: active`.
3. Find the task that cannot proceed and set `status: blocked` with a receipt:

```yaml
receipt:
  result: blocked
  summary: "Need credentials for production database before T007 can run verify."
  remaining_blockers:
    - "PROD_DB_URL env var required"
```

4. Create or activate the next safe local task that can proceed without the missing resource.

**If that doesn't work:** Run the checker after editing to confirm the board is consistent:

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

---

## `/goal` completes too early

**Cause:** The goal oracle signal is a placeholder (`<...>`, `tbd`, `unknown`, or similar). The PM finds verification clean and declares completion before the real owner outcome is proven.

**Fix:**

1. Open `docs/goals/<slug>/state.yaml` and set a concrete `goal.oracle.signal`:

```yaml
goal:
  oracle:
    signal: "npm test -- test/billing/router.test.ts exits 0 and all assertions pass"
```

2. Run the checker to surface oracle warnings before the next `/goal` run:

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

**If that doesn't work:** Confirm the final Judge receipt includes `full_outcome_complete: true` mapped explicitly to the oracle signal. A receipt with `decision: complete` but `full_outcome_complete: false` does not close the goal.

---

## Worker receipt fails `check-goal-state`

**Cause:** The most common cause is `changed_files` listing a file that does not match any pattern in the task's `allowed_files`. The validator also flags done Worker receipts with non-passing command statuses.

**Fix for `changed_files` mismatch:**

1. Open the Worker task in `state.yaml`.
2. Add the missing file or pattern to `allowed_files` before running the Worker again.
3. If the receipt was already written, either remove the out-of-scope file from `changed_files` or expand `allowed_files` and re-run the check.

**Fix for non-passing commands:**

Update the failing command entry once the issue is resolved:

```yaml
commands:
  - cmd: npm test -- test/billing/router.test.ts
    status: pass
```

**If that doesn't work:** If a verify command is legitimately expected to fail (for example, a known-flaky integration test), record the rationale in `summary` and create a Judge task to make an explicit accept/reject decision.

---

## Board not updating live

**Cause:** The EventSource connection between the browser and the GoalBuddy hub dropped. The board watches only `state.yaml` and `notes/`; changes to other files do not trigger updates.

**Fix:**

1. Look for the green "Live" dot in the board header. If absent, the connection dropped.
2. Refresh the browser tab. This reconnects the EventSource.
3. If frozen after refresh:

```bash
npx goalbuddy board docs/goals/<slug>
```

**If that doesn't work:** Confirm the hub process is still running at `http://127.0.0.1:41737/api/boards`. If that fails, the hub process exited — restart it.

---

## `check-goal-state` reports `legacy v1 goal state detected`

**Cause:** `state.yaml` does not have `version: 2`, or it contains v1-only fields: `gate:`, `artifact_policy:`, `active_unit:`, or references to `units/`, `artifacts/`, or `evidence.jsonl`.

**Fix:** Create a fresh goal with the current version of `goal-prep`. To migrate an existing board manually:

1. Add `version: 2` at the top of `state.yaml`.
2. Remove the `gate:` section entirely; add a `rules:` section if needed.
3. Remove `artifact_policy:` if present.
4. Rename `active_unit:` to `active_task:`.
5. Confirm no paths reference `evidence.jsonl`, `units/`, or `artifacts/`.
6. Run the checker:

```bash
node <skill-path>/scripts/check-goal-state.mjs docs/goals/<slug>/state.yaml
```

**If that doesn't work:** Migrate one section at a time and run the checker after each change to isolate remaining v1 fields.
