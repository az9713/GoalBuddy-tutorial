# GoalBuddy 0.3.7: Goalmaxxed

![GoalBuddy v0.3.7 release: Goalmaxxed.](https://raw.githubusercontent.com/tolibear/goalbuddy/v0.3.7/internal/assets/goalbuddy-v0.3.7-release.png)

Release date: 2026-05-19

Goalmaxxed is the release where GoalBuddy stops trying to become a workflow catalog and commits to one sharper job:

```text
Give /goal enough pressure that it keeps working until the original outcome is actually true.
```

This release is heavily inspired by the Codex maxxing playbook: keep the goal visible, preserve context in local files, use subagents deliberately, demand evidence, and resist the temptation to declare victory because a plausible slice finished.

Update with:

```bash
npx goalbuddy update
```

## The Headline

GoalBuddy now centers the native `/goal` loop around five small ideas:

- **Intent**: capture what the owner actually wants.
- **Oracle**: define the observable signal that proves the outcome is real.
- **Surface**: keep one local board visible while the run moves.
- **Loop**: Scout maps facts, Judge chooses the largest safe useful slice, Worker completes the whole slice.
- **Proof**: final completion requires receipts mapped back to the oracle.

That is the product. Everything else got judged against that loop.

## Goal Pressure

GoalBuddy now treats the goal oracle as first-class state. A serious goal needs an observable signal before the board can pretend it knows what done means:

- a passing test suite
- a browser walkthrough
- a demo transcript
- a generated artifact
- a benchmark
- a source-backed answer
- a release check
- a final human decision

No oracle, no serious goal.

The checker also rejects weak final completion. A goal should not be marked done just because the active task ended. Done means a final Judge or PM audit records that the receipts and verification satisfy the oracle.

## Larger Useful Slices

This release sharpens the slice policy:

```text
Safe does not mean small. Safe means bounded, explicit, verified, and reversible.
```

GoalBuddy now pushes Judge and Worker toward the largest safe useful slice: a working screen, working API path, backend vertical slice, real bug fix, data pipeline step, or milestone review.

The board warns when it sees micro-slicing: helper files, contracts, proof notes, or tiny prep tasks that are safe but do not move the owner outcome.

## Built-In Local Board

The local board is now a core surface, not an extension.

The bundled surface lives at:

```text
goalbuddy/surfaces/local-goal-board/
```

It remains the default way to watch a GoalBuddy run: active task, blocked state, receipts, verification status, subgoals, and board switching all point back to the same `state.yaml` truth.

## No Extension Catalog

GoalBuddy no longer ships a public extension catalog.

The old catalog made the product look bigger while making the core loop blurrier. Goalmaxxed chooses the smaller invariant:

```text
GoalBuddy prepares and pressures /goal runs. Custom integrations are ordinary repo work.
```

If a team wants a GitHub, Linear, Slack, or release integration, they should prepare a concrete implementation plan in their repo and build it as normal software. GoalBuddy should not install arbitrary workflow packs as a side channel.

## Simpler Public Surface

The public copy now says what GoalBuddy actually does:

- prepares `/goal`
- writes `goal.md` and `state.yaml`
- creates a goal oracle
- opens a local board
- keeps Scout/Judge/Worker handoffs receipt-shaped
- prevents early completion
- leaves custom integrations outside the core

The Codex and Claude Code plugin manifests are aligned with the package description, and the test suite now checks that the Claude manifest stays in sync.

## Release Boundaries

This release intentionally does not add:

- an extension marketplace
- automatic parallel-agent spawning
- hosted board state
- automatic receipt application
- UI controls that mutate board state
- a replacement for native `/goal`

GoalBuddy stays local, file-backed, and boring in the parts that should be boring.

## Package Notes

This release updates:

- npm package version: `0.3.7`
- Codex plugin version: `0.3.7`
- Claude Code plugin version: `0.3.7`
- package contents to include `goalbuddy/surfaces/`
- mirrored GoalBuddy skill files under `plugins/goalbuddy/skills/goalbuddy/`

Before publishing, verify:

```bash
npm run check
npm run pack:dry-run
node internal/cli/check-publish-version.mjs
```
