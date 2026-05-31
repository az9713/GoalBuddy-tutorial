# GoalBuddy

GoalBuddy is an npm CLI tool that gives Codex and Claude Code a structured operating loop for long `/goal` coding tasks — a charter, a live board, a proof loop, and durable receipts so runs can pause, resume, and complete without losing their place.

---

| Section | What's inside |
|---|---|
| [What is GoalBuddy?](overview/what-is-goalbuddy.md) | The problem it solves, the mental model, and architecture overview |
| [Key concepts](overview/key-concepts.md) | Alphabetical glossary of all GoalBuddy terms |
| [Prerequisites](getting-started/prerequisites.md) | Node.js version, supported agents, and verification steps |
| [Quickstart](getting-started/quickstart.md) | Install, goal prep, board up, and first `/goal` run |
| [Onboarding](getting-started/onboarding.md) | Mental model walkthrough and a realistic scenario from start to finish |
| [The loop](concepts/the-loop.md) | Intent → Oracle → Surface → Loop → Proof explained |
| [The board](concepts/the-board.md) | goal.md, state.yaml, tasks, and receipts |
| [Agents](concepts/agents.md) | Scout, Judge, Worker, and PM — roles, contracts, receipts |
| [The oracle](concepts/oracle.md) | What makes a good goal oracle and why it is required |
| [Local board](concepts/local-board.md) | The live board server, hub model, and settings |
| [Prepare a goal](guides/prepare-a-goal.md) | Running goal-prep, answering diagnostic questions |
| [Run a goal](guides/run-a-goal.md) | Executing `/goal`, continuing and completing a run |
| [Write a receipt](guides/write-a-receipt.md) | Receipt formats for Scout, Judge, Worker, and PM |
| [Use parallel agents](guides/use-parallel-agents.md) | `goalbuddy parallel-plan` and safe concurrent work |
| [CLI reference](reference/cli.md) | Every `npx goalbuddy` command, flag, and option |
| [state.yaml reference](reference/state-yaml.md) | Full schema for the board truth file |
| [goal.md reference](reference/goal-md.md) | Every field in the charter template |
| [System design](architecture/system-design.md) | Components, data flows, and key design decisions |
| [Troubleshooting](troubleshooting/common-issues.md) | Most common failures and exact fixes |

New here? Start with the [onboarding guide](getting-started/onboarding.md).
