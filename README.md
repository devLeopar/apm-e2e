# APM E2E

[![License: MPL-2.0](https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg)](https://opensource.org/licenses/MPL-2.0)

*A custom APM adaptation that runs E2E scenarios inline and turns the passing ones into a Manager-orchestrated regression suite.*

## What is APM E2E?

APM E2E is a custom adaptation of [Agentic Project Management (APM)](https://github.com/sdi2200262/agentic-project-management) for projects that ship with end-to-end behavioral coverage — web (Playwright, Cypress), mobile (Maestro, Detox), or API (Playwright, supertest). It layers two changes on top of upstream APM v1 and otherwise leaves the framework unchanged:

1. **Inline Worker E2E execution.** Workers run E2E scenarios in their own context against the feature they just built — no separate validator subagent, no dispatch toggle. The agent that writes the feature is the one that runs the behavioral checks against it.
2. **E2E Test Corpus + Manager regression.** Passing scenarios are persisted as runner-native test code (Maestro YAML, Playwright spec, etc.) into the project tree with a `# APM source: Task N.M` traceability header. After each E2E-passing Task, the Manager runs the full corpus during Task Review — before marking the Task Done — and triages any failure into one of four branches.

It is for projects where "Done" should mean "the behavior still works alongside everything that worked before." Internal libraries, throwaway prototypes, and codebases without a runtime to exercise can opt out by declaring `## E2E Testing Policy: never` in the Spec, in which case the entire layer is inert and APM E2E behaves identically to APM v1.

## Installation

Navigate to your project directory, initialize through the `agentic-pm` CLI:

```bash
npm install -g agentic-pm
apm custom -r devLeopar/apm-e2e --tag v1.0.1-e2e-2
```

Open your AI assistant and start planning:

```
/apm-1-initiate-planner
```

The Planner collaborates with you through project discovery and creates the planning documents. During Round 2 of Context Gathering it asks about E2E policy, corpus location, and regression cadence; the Spec records the answers; the Manager orchestrates persistence and regression at runtime. Once approved, open a new chat and run `/apm-2-initiate-manager` to begin the Implementation Phase.

## How It Works

APM E2E inherits APM v1 unchanged — same three agents, same two phases, same artifacts, same commands. The E2E layer adds:

- **Planner — three new questions in Round 2.** Whether to run E2E (`always` / `task-tagged` / `never`), where the corpus lives (e.g. `.maestro/`, `e2e/playwright/`), and how often the Manager runs regression (`per-task` / `per-stage` / `manual-only`).
- **Worker — inline scenario execution and test persistence.** When a Task carries an `## E2E Validation` block, the Worker assembles a Brief (target, scenarios, acceptance, tool hints, iteration budget), runs the scenarios with the runner declared in the Spec, iterates on application-side fixes when scenarios fail, and — when a `### Persistence` sub-block is present — writes the passing scenarios out as runner-native test files into the corpus path with a `# APM source: Task N.M` header.
- **Manager — regression run during Task Review.** After accepting an E2E-passing Task and before marking it Done in the Tracker, the Manager runs the full corpus per the Spec's cadence. Failures are triaged into four branches: *consumer broke producer's feature* (open follow-up to fix the consumer), *producer's test is stale* (open follow-up to update the test), *consumer adapts to existing contract* (rework the current Task), or *ambiguous* (escalate to the User).

The four-branch triage uses Manager arbitration thresholds analogous to upstream Planning Document Modification authority: low-confidence ambiguity always escalates rather than guessing.

The Spec records the policy in three knobs the Planner writes during planning:

```markdown
## E2E Testing Policy: always | task-tagged | never

## E2E Test Corpus
- Path: .maestro/
- Runners: maestro, playwright
- Cadence: per-task | per-stage | manual-only
```

`per-task` (default) runs the corpus after every E2E-passing Task. `per-stage` defers regression to Stage Verification, trading early detection for fewer runs. `manual-only` runs only when the User explicitly asks — useful for prototyping phases where the corpus is changing too fast to be a useful gate.

## Trade-offs

APM E2E trades Worker context space and Manager review time for behavioral certainty:

- Worker context fills faster — test runner output (Playwright traces, Maestro device logs, stack traces from failed assertions) lands directly in the conversation. Workers iterate against full output instead of a subagent's compressed summary. This is the deliberate cost of dropping the validator subagent.
- Task Review takes longer — the Manager runs the full corpus before marking each Task Done. Cost grows linearly with corpus size; the cadence policy exists to bound it.
- The project tree gains a tests directory that needs to be maintained — stale tests against deleted features need to be removed by hand or by follow-up Tasks.

If your project does not have a runtime to exercise (an internal library, a documentation site, a research notebook), declare `## E2E Testing Policy: never` and APM E2E behaves identically to APM v1 — no Brief, no persistence, no regression run.

## Commands

APM E2E keeps the v1 command surface unchanged. The E2E layer is opt-in via the Spec, not new commands.

| # | Command | Agent | Purpose |
|---|---------|-------|---------|
| 1 | `/apm-1-initiate-planner` | Planner | Planning Phase |
| 2 | `/apm-2-initiate-manager` | Manager | Implementation Phase |
| 3 | `/apm-3-initiate-worker` | Worker | Worker initialization |
| 4 | `/apm-4-check-tasks` | Worker | Task Bus check |
| 5 | `/apm-5-check-reports` | Manager | Report Bus check |
| 6 | `/apm-6-handoff-manager` | Manager | Manager Handoff |
| 7 | `/apm-7-handoff-worker` | Worker | Worker Handoff |
| 8 | `/apm-8-summarize-session` | Standalone | Session summary and archival |
| 9 | `/apm-9-recover` | Manager or Worker | Reconstruct context after compaction |

## Documentation

For the full APM workflow documentation, see [agentic-project-management.dev](https://agentic-project-management.dev). APM E2E shares APM's core concepts (planning documents, Stages, Tasks, Memory, Handoff) with the inline-execution and regression layers added on top.

## Contributing

Contributions are welcome. Report bugs or suggest features via [GitHub Issues](https://github.com/devLeopar/apm-e2e/issues).

## License

Licensed under the **Mozilla Public License 2.0 (MPL-2.0)**. See [LICENSE](LICENSE) for full details.

Based on [Agentic Project Management](https://github.com/sdi2200262/agentic-project-management) by [CobuterMan](https://github.com/sdi2200262).
