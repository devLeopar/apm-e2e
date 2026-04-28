---
name: apm-e2e-validation
description: Standards for end-to-end behavior validation - policy, scenario rendering, E2E Test Brief format, inline execution, test code persistence, and regression run.
---

# APM {VERSION} - E2E Validation Skill

## 1. Overview

**Reading Agents:** Planner, Manager, Worker

This skill defines how end-to-end behavior validation operates across the APM workflow. The Planner consults it when writing acceptance criteria and declaring policy in the Spec during Work Breakdown. The Manager consults it when rendering scenarios into Task Prompts, presenting recommendations under `ask-per-task` policy, and running corpus regression after Task Review. The Worker consults it when assembling the E2E Test Brief from a Task Prompt's E2E Validation section, executing the scenarios inline at Task Validation, and persisting passing scenarios to the project's E2E Test Corpus.

### 1.1 Outputs

- *Spec's E2E Testing Policy section:* Written by the Planner during Work Breakdown.
- *Spec's E2E Test Corpus section:* Written by the Planner during Work Breakdown when E2E policy is not `never`.
- *Task Prompt's E2E Validation section:* Rendered by the Manager during Task Prompt Construction.
- *E2E Test Brief:* Assembled by the Worker during Task Execution as the execution checklist.
- *`e2e_result`:* Structured outcome captured by the Worker and embedded in the Task Log per §5.3.
- *Persisted test files:* Written by the Worker into the corpus path per §6, forming the project's regression suite.
- *`regression_result`:* Structured outcome captured by the Manager during Task Review and embedded in the Tracker per §7.3.

---

## 2. E2E Testing Policy

The Spec's `## E2E Testing Policy` section declares one of three values, captured by the Planner from Context Gathering:

- *`auto`:* E2E runs by default on Tasks with observable user-facing behavior. Pure refactors, internal utilities, and Tasks with no user-visible output are exempt automatically. The Manager still presents what will be validated before dispatch - the User can opt out per-Task without changing project policy.
- *`ask-per-task`:* The Manager asks the User before each dispatch, presenting a recommendation based on Task nature. Typical recommendation logic: UI-heavy or behavioral Tasks warrant E2E; refactors, docs, tooling typically do not. Recommendation includes context-cost awareness - simple Tasks with trivial acceptance don't justify E2E execution overhead in the Worker's context.
- *`never`:* E2E validation is not part of the workflow. No E2E sections appear in Task Prompts or Task Logs. MCP Dependencies declared for E2E-specific purposes are also omitted.

The Planner proposes the policy during Context Gathering based on project nature (user-facing app → lean toward `auto` or `ask-per-task`; internal library → lean toward `never`) and User preference. Policy is changed mid-project through a Spec update per WORKFLOW.md §3.4 Document Modification.

---

## 3. Three-Level Scenario Rendering

E2E content flows through three rendering levels, each owned by a different agent. Each level adds information the next level needs without duplicating the previous.

**Level 1 - Planner writes product-level acceptance criteria** in the Task's Plan Validation field during Work Breakdown. Example:

> User can sign in with valid credentials and lands on the dashboard. Invalid credentials show a clear error message. The sign-in flow completes within 3 seconds on a reasonable network.

Criteria describe product behavior in natural prose. No framework names, no selectors, no DOM paths, no tool references.

**Level 2 - Manager renders test-executable scenarios** into the Task Prompt's E2E Validation section during Task Prompt Construction. Example:

```markdown
## E2E Validation

### Scenarios
1. **Happy path sign-in**
   - Navigate to /login
   - Enter valid email and password in respective fields
   - Submit the form
   - Expect: redirect to /dashboard within 3s, heading text containing "Welcome" visible

2. **Invalid credentials error**
   - Navigate to /login
   - Enter valid email, wrong password
   - Submit the form
   - Expect: error message "Invalid credentials" visible within 1s, URL remains /login

### Acceptance
- Both scenarios pass their individual checks
- No console errors observed across either scenario
- Network requests complete without 5xx responses

### Artifact Path
.apm/e2e-artifacts/stage-<NN>/task-<NN>-<MM>/

### Persistence
- Path: `e2e/playwright/`
- Format: Playwright `.spec.ts`
- Naming: `<feature-area>-<scenario>.spec.ts`
- Header: `// APM source: Task <N.M> - <Title>`
```

Scenarios describe observable user actions with pass conditions. Selectors remain unspecified - the Worker resolves them at runtime from the UI. Artifact path follows the convention in §9. The Persistence block is included only when the Spec declares an `## E2E Test Corpus` section - it tells the Worker where to write passing scenarios as runner-native test code per §6 Test Code Persistence. When the Spec has no corpus section, the Persistence block is omitted and no test code is persisted.

**Level 3 - Worker enriches with runtime environment** when assembling the E2E Test Brief. The Worker adds: target URL or app identifier, launch commands if the target is not already running, ready signals to wait for, available MCPs, preferred runner and fallback order, iteration budget. The Brief is the Worker's structured execution checklist - it captures everything needed to run the scenarios end-to-end without re-deriving runtime details mid-execution.

---

## 4. E2E Test Brief Format

The Brief is the structured spec the Worker assembles from the Task Prompt's E2E Validation section before executing. It consolidates Manager-rendered scenarios with the runtime environment so execution proceeds from a single coherent reference rather than re-reading the Task Prompt mid-run.

**Brief Schema:**

```yaml
target:
  type: web | mobile-expo | mobile-native | backend-api
  url: <string, for web and backend-api>
  app_identifier: <string, for mobile>
  simulator: <string, for mobile>
  port: <int, when applicable>
  launch_commands:
    - <shell command>
  ready_signal: <substring to match in launch output>
scenarios:
  - name: <string>
    steps:
      - <natural language user action>
    acceptance:
      - <observable condition>
acceptance:
  - <cross-scenario condition>
artifact_path: <project-relative directory>
tool_hints:
  preferred_runner: <runner name>
  fallback_order: [<runner>, ...]
  available_mcps: [<mcp name>, ...]
iteration_budget: <int, default 3>
```

**Field Descriptions:**

- *`target.type`:* Runtime family - `web`, `mobile-expo`, `mobile-native`, `backend-api`. Determines default runner selection.
- *`target.launch_commands`:* Commands to start the target if not already running. The Worker runs these before the scenario loop and waits for the ready signal; alternatively the Worker starts the target separately and sets `launch_commands` to empty.
- *`target.ready_signal`:* Substring to wait for in launch output before proceeding (e.g., "Bundled in 8000ms", "ready on port 3000").
- *`scenarios[].steps`:* Natural language user actions, one per step. The Worker translates these into runner-specific commands during execution.
- *`scenarios[].acceptance`:* Observable pass conditions for the scenario. Verifiable without access to application internals.
- *`acceptance`:* Cross-scenario conditions that must hold after all scenarios complete (no console errors, performance thresholds).
- *`artifact_path`:* Pre-created directory where screenshots, traces, and videos are saved. The Worker creates this directory before the scenario loop.
- *`tool_hints.preferred_runner`:* Runner to try first. When absent, falls back to §5.1 Tool Selection default order for the target type.
- *`tool_hints.available_mcps`:* MCPs the Worker has confirmed are available (from the Task Prompt's MCP context or the project Spec).
- *`iteration_budget`:* Maximum test-side retry attempts per scenario (default 3). Applies only to test-side adjustments (selectors, timing) - application-side failures exit to the standard correction loop per `{GUIDE_PATH:task-execution}` §3.5.

The Worker omits fields that do not apply to the target type. The Brief lives in the Worker's working context only - it is not written to a separate file.

---

## 5. Execution

The Worker executes the E2E Test Brief inline during Task Validation per `{GUIDE_PATH:task-execution}` §3.4. Execution stays in the Worker's context window - test runner output, screenshots, and retry traces accumulate alongside other Task work.

### 5.1 Tool Selection

Select a runner in this order, using the first that is both available and suitable for the target:

1. *Runner explicitly preferred in `tool_hints.preferred_runner`* (e.g., "Playwright MCP for web", "Maestro for React Native").
2. *Platform-native runner for the target type* per §8 Runtime Variants:
   - *Web:* Playwright MCP → Playwright direct → Puppeteer direct.
   - *React Native (Expo or bare):* Maestro → Detox → Expo web export with Playwright.
   - *Native iOS:* XCUITest via shell.
   - *Native Android:* Espresso via shell.
   - *Backend API:* Shell with curl or httpie for request-based scenarios; Playwright API request context for HTTP test flows.
3. *Report `blocked` status* with the missing dependency listed when no runner fits.

When multiple runners could work, prefer MCP-based runners - they integrate with the tool-use loop and produce structured output directly. Shell-based runners require parsing test output, which is error-prone and consumes more context.

### 5.2 Scenario Execution

For each scenario in the Brief, in order:

1. Perform the user actions using the selected runner, as described in the scenario's steps. Capture a screenshot at each significant state transition and on any failure.
2. Check the scenario's acceptance conditions.
3. On acceptance pass, record the scenario as `pass` with the iteration count consumed.
4. On acceptance fail, enter the test-side correction loop within the same scenario:
   - Inspect UI hierarchy, recent screenshots, and any console or device output.
   - Determine whether the failure is test-side (wrong selector, premature assertion, race condition) or application-side (actual behavior mismatch).
   - If test-side and within `iteration_budget`, adjust the test approach and retry. Test-side adjustments do not change application code.
   - If application-side, stop iterating immediately, mark the scenario `fail` with a concrete `failure_reason` (what was observed vs what was expected, relevant artifact paths), and exit the scenario loop to the standard correction loop per `{GUIDE_PATH:task-execution}` §3.5. After applying an application-side fix, re-run all scenarios from the start.

After all scenarios complete, check cross-scenario acceptance conditions (no console errors, performance thresholds). Capture final state. When `e2e_result.status: pass` and the Spec declares an `## E2E Test Corpus` section, persist the passing scenarios to the corpus per §6 Test Code Persistence.

If the target fails to start within a reasonable timeout, or the selected runner cannot reach the target, return `status: blocked` with the launch error or missing dependency captured. Do not continue with degraded execution.

### 5.3 E2E Result Format

Capture execution outcomes in this structure and embed verbatim in the Task Log's E2E Validation section per `{GUIDE_PATH:task-logging}` §4.1.

```yaml
e2e_result:
  status: pass | fail | blocked
  runner: <selected runner name>
  scenarios:
    - name: <scenario name from Brief>
      status: pass | fail
      attempts: <int>
      artifacts:
        - <project-relative path>
      failure_reason: <string, present when fail>
  cross_scenario_acceptance:
    status: pass | fail
    issues: [<string>, ...]
  mcp_used: [<MCP name>, ...]
  total_duration_ms: <int>
  failure_summary: <string, present when status is not pass>
```

**Field Descriptions:**

- *`status`:* Overall verdict. `pass` only when all scenarios pass and cross-scenario acceptance passes. `fail` when at least one scenario or acceptance condition fails. `blocked` when execution could not proceed (missing runner, target not reachable, missing MCP).
- *`runner`:* The runner actually used (after tool selection resolution).
- *`scenarios[].status`:* Per-scenario pass/fail.
- *`scenarios[].attempts`:* How many test-side attempts were made before the final verdict.
- *`scenarios[].artifacts`:* All artifacts for this scenario - screenshots at state transitions, trace files, videos.
- *`scenarios[].failure_reason`:* Present when `status: fail`. Observed vs expected, with artifact references.
- *`cross_scenario_acceptance`:* Result of cross-cutting conditions (console errors, performance thresholds, etc.).
- *`mcp_used`:* MCPs that were called during execution.
- *`total_duration_ms`:* Wall-clock duration of the full run.
- *`failure_summary`:* One-paragraph synthesis when the verdict is not pass. The Manager reads this during Task Review to decide follow-up direction.

When an application-side fix triggers a re-run per §5.2, replace the previous `e2e_result` with the latest run's outcome. The Task Log captures the final verdict; intermediate re-run iterations may be noted in prose alongside the structured block.

---

## 6. Test Code Persistence

When the Spec declares `## E2E Test Corpus`, the Worker persists passing scenarios as executable test code in the project tree after `e2e_result.status: pass`. Persisted tests form the regression suite that the Manager runs against subsequent Tasks per §7. This turns ad-hoc per-Task validation into an accumulating safety net for future work.

### 6.1 When to Persist

Persistence runs once per Task, after the Worker's own scenarios pass:

- *On first pass* (clean run, no application-side iteration): persist immediately and proceed to Task Completion.
- *After application-side correction:* persist after the final passing run that reflects the corrected behavior.
- *On `fail` or `blocked`:* do not persist - the corpus contains only canonical passing tests.

When a follow-up Task uses the same `log_path` as the original (per `{GUIDE_PATH:task-assignment}` §2.3), the Worker overwrites the previous test files for that Task's feature area.

### 6.2 Storage Convention

The Spec's `## E2E Test Corpus` section declares the corpus location and runner formats. Default paths by primary runtime when the project has no pre-existing test infrastructure:

| Runtime | Default Path | Format |
|---------|--------------|--------|
| Web (Playwright) | `e2e/playwright/` | `.spec.ts` |
| React Native Expo (Maestro) | `.maestro/` | `.yaml` |
| React Native bare (Detox or Maestro) | `.maestro/` or `e2e/detox/` | `.yaml` or `.test.ts` |
| Backend API (Playwright request) | `e2e/api/` | `.spec.ts` |

When the project already has test infrastructure (e.g., existing `tests/e2e/`, `cypress/e2e/`, `.maestro/`), the Spec captures the existing path and the Worker uses it. The corpus integrates with the project's real test setup rather than creating a parallel one.

### 6.3 File Naming

Filenames are semantic, reflecting the feature area and scenario - they live in the project tree and are read by future engineers regardless of APM. Pattern: `<feature-area>-<scenario-name>.<ext>`. Examples:

- `auth-signin.yaml` (Maestro)
- `nav-tabs.spec.ts` (Playwright)
- `api-create-user.spec.ts` (Playwright API)

The Worker chooses meaningful names. When a Task contains multiple scenarios for one feature area, group into one file (Maestro multi-flow YAML, Playwright `test.describe` block) or split into multiple files - whichever matches the runner's natural unit.

### 6.4 Header Comment

Each persisted test file starts with a header comment for traceability:

```
# APM source: Task <N.M> - <Task Title>
# Scenario: <scenario name from Brief>
```

Comment syntax follows the runner's convention (`#` for YAML and shell, `//` for TS/JS). The Manager parses the `APM source:` line during regression triage to identify the producing Task per §7.4. Workers must not omit this header - regression triage depends on it.

### 6.5 Translation from Brief

The persisted test code expresses the Brief's `scenarios[].steps` and `acceptance` in the runner's native syntax. The Worker translates the natural-language steps into runner commands, parameterizing what was hardcoded during inline execution (timeouts, selectors, assertion thresholds). The result is reproducible: re-running the persisted file against the running target should pass without further adjustment.

When the inline run required test-side iterations (selector refinement, timing adjustments), the persisted file reflects the final stable version - not the initial guess. This way regression runs do not re-discover the same flakiness.

### 6.6 Logging

The Worker logs persisted file paths in the Task Log's `## Test Persistence` section per `{GUIDE_PATH:task-logging}` §4.1. The Manager reads this section during Task Review to know what entered the corpus before running regression per §7.

---

## 7. Regression Run

The Manager runs the corpus regression after each E2E-passing Task - or per Stage when Spec policy is set to `per-stage`. The procedure verifies that the just-completed Task did not break any previously-persisted scenario.

### 7.1 Trigger and Cadence

The Spec's `## E2E Test Corpus` section declares the cadence:

- *`per-task` (default):* Manager runs the full corpus after Task Review confirms the Worker's E2E passed and tests were persisted. Catches regression immediately, blocks Done until the regression result is processed.
- *`per-stage`:* Manager runs the corpus once at Stage end as part of Stage Verification per `{GUIDE_PATH:task-review}` §2.8. Lighter per-Task overhead, regression risk accumulates within the Stage.
- *`manual-only`:* Manager does not run regression automatically. The User can request a regression run at any time; the Manager runs it on demand.

When the corpus is empty (first Task with E2E in the project), regression is a no-op - the Manager notes "corpus empty, no regression to run" and proceeds. Regression runs only when the just-completed Task itself produced a passing E2E - if the Task had no E2E or E2E failed, no regression follows.

### 7.2 Execution

The Manager runs regression in their own context, similar to the Worker's per-Task execution per §5:

1. List all test files under the corpus path declared in the Spec.
2. Resolve the runner per §5.1 Tool Selection based on the file extension and target runtime. The corpus may contain mixed runners (e.g., Maestro YAML + Playwright TS) - run each set with its respective tool.
3. For each test file:
   - Start the target if not already running (same launch logic as the Brief's `target.launch_commands`).
   - Execute the file against the running target.
   - Capture pass/fail status, runtime, and any failure output to the test's artifact directory under `.apm/e2e-artifacts/regression/<datetime>/`.
4. After all tests run, assemble the structured `regression_result` per §7.3.

The Manager runs regression itself rather than dispatching to a Worker - this keeps the regression cycle in the coordination context where triage happens, avoids spinning up Worker chats just to re-run existing tests, and keeps Worker context budgets focused on new feature work.

### 7.3 Regression Result Format

```yaml
regression_result:
  status: pass | fail | blocked | empty
  cadence: per-task | per-stage | manual-only
  trigger: task-<N.M> | stage-<NN> | user-requested
  tests_run: <int>
  tests_passed: <int>
  tests_failed: <int>
  failures:
    - file: <project-relative path>
      producer_task: <N.M>
      failure_reason: <string>
      artifacts:
        - <project-relative path>
  total_duration_ms: <int>
  triage_notes: <string, present when status: fail>
```

**Field Descriptions:**

- *`status`:* `pass` when all tests pass. `fail` when at least one fails. `blocked` when execution could not proceed (missing runner, target not reachable). `empty` when corpus has no tests.
- *`cadence`:* The cadence policy that triggered this run.
- *`trigger`:* What triggered this regression run - the Task ID for `per-task`, Stage number for `per-stage`, or `user-requested` for manual invocation.
- *`failures[].producer_task`:* The Task that originally produced this test, parsed from the file's `# APM source:` header per §6.4.
- *`triage_notes`:* Manager's notes on triage outcome per §7.4 - which branch was taken and the follow-up direction.

The Manager records `regression_result` in the Tracker's Working Notes per `{GUIDE_PATH:task-review}` §2.7 with the triggering Task ID. When the regression result rolls into a Stage outcome (per-stage cadence), the result is also summarized in the Stage summary per `{GUIDE_PATH:task-review}` §2.6.

### 7.4 Failure Triage

When `regression_result.status: fail`, the Manager triages each failing test against the just-completed Task to determine the corrective action. The producer Task is identified from the test file's header per §6.4; the consumer Task is the just-completed Task.

Three branches:

- *Consumer broke producer's feature:* The current Task's changes inadvertently broke a previously-passing scenario. Follow-up Task to the current Worker per `{GUIDE_PATH:task-assignment}` §3.4 with regression details and integration guidance to fix the breakage. The current Worker re-runs their own E2E and re-persists; the Manager re-runs regression after.
- *Producer's test is stale:* The current Task legitimately changed the behavior the old test asserts (intentional design evolution). Follow-up Task to the producer Worker to update the test and any code the test depends on, or the Manager updates the test inline if the change is small and clear per `{GUIDE_PATH:task-review}` §2.3 Planning Document Modification Standards.
- *Consumer adapts:* The current Task should adapt its approach to fit the existing feature contract rather than break it. Follow-up Task to the current Worker with adaptation guidance. The producer's test stays unchanged.
- *Ambiguous:* Present the regression to the User with the Manager's reading of the failure and the three options. User decides the direction.

Triage uses the same authority threshold as planning document modification: Manager handles unambiguous cases directly; ambiguous or scope-significant cases route to the User. The Manager records the triage decision in `triage_notes` and as a Working Note per `{GUIDE_PATH:task-review}` §2.7.

When a regression follow-up loops back through Task Execution, the corrected Task re-runs its own E2E and re-persists test code per §6, then the Manager re-runs regression. Bounded by the standard correction loop budget per `{GUIDE_PATH:task-execution}` §3.5 - exhausting it presents the situation to the User.

---

## 8. Runtime Variants

**Web apps.** Prefer Playwright MCP when available - it integrates with the tool-use loop and returns structured snapshots. Target is `http://localhost:<port>` for local dev; launch command is typically `npm run dev` or `pnpm dev`; ready signal is the port-listening log line (e.g., "Local: http://localhost:3000"). Artifacts are PNG screenshots and Playwright trace files. Persisted format: Playwright `.spec.ts` files.

**React Native Expo.** Prefer Maestro - it handles native gestures on Expo Go across iOS and Android with the same flow definitions. Target is the app identifier (e.g., `host.exp.Exponent` for Expo Go) and simulator name. Launch via `npm run ios` or `npm run android`; ready signal is Metro bundler's "Bundled Xms" output or the app showing on the simulator. When Maestro is not available, Expo web export (`npx expo export --platform web` then serve) with Playwright is a fallback - limited to flows that don't use native-only APIs (camera, biometrics, notifications). Persisted format: Maestro flow YAML.

**React Native bare.** Same as Expo but launch via `npx react-native run-ios` or `npx react-native run-android`. Detox is a viable alternative to Maestro when the project has Detox already set up.

**Backend API.** Target is the API base URL. Scenarios describe request flows - sequences of HTTP calls with expected status codes and response shapes. Playwright's `request` context handles this cleanly; shell with `curl` or `httpie` works as a fallback for simple flows. Persisted format: Playwright `.spec.ts` API test files.

When the project uses a runtime not covered here, the Worker notes this in the Brief's `tool_hints` with specific runner guidance and adapts tool selection per §5.1.

---

## 9. Artifact Path Convention

The Manager generates the artifact path when rendering Level 2 scenarios using this pattern:

```
.apm/e2e-artifacts/stage-<NN>/task-<NN>-<MM>/
```

For regression runs, the Manager uses:

```
.apm/e2e-artifacts/regression/<datetime>/
```

The Worker creates the per-Task directory before the scenario loop (via `mkdir -p` or equivalent) and saves all scenario artifacts within this path. The Manager creates the regression directory before each regression run. The Task Log references artifacts by their project-relative paths.

Artifacts are not tracked in version control by default - `.apm/` is in `.gitignore` per WORKFLOW.md §3.3 Rules. When the User wants artifacts committed (for PR review or CI evidence), they adjust `.gitignore` manually. Persisted test files in the project tree (per §6.2) are tracked normally - they live outside `.apm/`.

---

**End of Skill**
