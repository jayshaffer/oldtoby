# Pipeline Orchestrator Requirements

## User Story 1: Nightly Pipeline Execution
As a product ideation operator, I want an autonomous nightly pipeline that advances every active idea by one stage,
so that ideas progress through the review gauntlet without manual intervention.

### Acceptance Criteria
- WHEN the cron schedule fires (default 02:00 Mountain), THE SYSTEM SHALL invoke `/run-pipeline` via `claude -p "/run-pipeline" --model sonnet`.
- WHEN `/run-pipeline` is invoked, THE SYSTEM SHALL read `config/pipeline.config.json` for schedule, thresholds, and model settings.
- WHEN a run log already exists for today's date in `logs/runs/`, THE SYSTEM SHALL skip the brainstorm phase but still process existing ideas in `pipeline/`.
- WHEN no run log exists for today, THE SYSTEM SHALL invoke `/brainstorm` as a subagent before processing ideas.

## User Story 2: Idea Dispatch by Status
As the pipeline orchestrator, I want to route each idea to the correct agent based on its frontmatter status,
so that ideas progress through the pipeline in the correct order.

### Acceptance Criteria
- WHEN scanning `pipeline/`, THE SYSTEM SHALL read `status` and `iteration` from each idea's `one-pager.md` frontmatter.
- WHEN an idea has `status: brainstormed`, THE SYSTEM SHALL spawn a subagent for `/agent-pm`.
- WHEN an idea has `status: prd-ready`, THE SYSTEM SHALL spawn a subagent for `/agent-market`.
- WHEN an idea has `status: market-ready`, THE SYSTEM SHALL spawn a subagent for `/agent-tech`.
- WHEN an idea has `status: tech-ready`, THE SYSTEM SHALL spawn a subagent for `/agent-cfo`.
- WHEN an idea has `status: cfo-ready`, THE SYSTEM SHALL spawn a subagent for `/agent-decision`.
- WHEN an idea has `status: promoted`, THE SYSTEM SHALL move the folder to `review/`.
- WHEN an idea has `status: killed`, THE SYSTEM SHALL move the folder to `trash/`.
- WHEN processing ideas, THE SYSTEM SHALL process in stage order (all `brainstormed` first, then `prd-ready`, etc.) to prevent double-processing within a single run.

## User Story 3: Verdict Reading and State Transitions
As the pipeline orchestrator, I want to read agent verdicts from `notes.md` and update idea state accordingly,
so that the filesystem remains the single source of truth for pipeline state.

### Acceptance Criteria
- WHEN a subagent completes, THE SYSTEM SHALL read the most recent entry in `notes.md` to extract `**Verdict:**` and `**Next status:**`.
- WHEN a verdict is received, THE SYSTEM SHALL update the `**Status:**` field in the idea's `one-pager.md` frontmatter.
- WHEN a verdict is received, THE SYSTEM SHALL increment the `**Iteration:**` field in the idea's `one-pager.md` frontmatter.
- WHEN an idea's `iteration` exceeds `max_iterations_per_idea` from config (default 5), THE SYSTEM SHALL invoke the decision agent with a circuit-breaker flag, forcing a promote-or-kill.

## User Story 4: Reconciliation Loops
As the pipeline orchestrator, I want to re-process ideas whose status changed due to delegation or send-back within the same run,
so that ideas can advance multiple steps in a single nightly execution.

### Acceptance Criteria
- WHEN any idea's status changed during the dispatch phase (delegation or send-back), THE SYSTEM SHALL re-scan `pipeline/` and re-process changed ideas.
- WHEN running reconciliation, THE SYSTEM SHALL repeat up to `max_reconciliation_loops` times (default 2).
- WHEN reconciliation loops are exhausted, THE SYSTEM SHALL leave remaining in-progress ideas for the next nightly run.

## User Story 5: Terminal Moves and Logging
As a product ideation operator, I want promoted ideas moved to `review/` and killed ideas moved to `trash/` with proper logging,
so that I can easily find actionable ideas and understand pipeline health.

### Acceptance Criteria
- WHEN an idea reaches `status: promoted`, THE SYSTEM SHALL move its folder from `pipeline/` to `review/`.
- WHEN an idea reaches `status: killed`, THE SYSTEM SHALL move its folder from `pipeline/` to `trash/`.
- WHEN an idea is promoted, THE SYSTEM SHALL append an entry to `logs/changelog.md` with title, iterations, agents passed, human-refined flag, elevator pitch, key concern, refinement history, estimated build timeline, and estimated monthly cost.
- WHEN terminal moves complete, THE SYSTEM SHALL update each idea's entry in `logs/idea-registry.md` with the final outcome.
- WHEN the run completes, THE SYSTEM SHALL write a pipeline snapshot and run summary to `logs/runs/{YYYY-MM-DD}.log.md`.
- WHEN the run completes, THE SYSTEM SHALL update `logs/metrics.json` with cumulative stats.
- WHEN ideas are promoted to `review/`, THE SYSTEM SHALL print `\a` to trigger a terminal bell notification.

## User Story 6: Manual Idea Injection
As a product ideation operator, I want to manually inject ideas into `pipeline/` and have them picked up automatically,
so that I can add my own ideas to the pipeline without special tooling.

### Acceptance Criteria
- WHEN scanning `pipeline/` and an idea has no entry in `logs/idea-registry.md`, THE SYSTEM SHALL append a registry entry with slug, title, problem, keywords, and current status before dispatching to the appropriate agent.
- WHEN a manually injected idea has `status: brainstormed`, THE SYSTEM SHALL route it to the PM agent like any brainstormed idea.

## User Story 7: Subagent Architecture
As the pipeline orchestrator, I want each agent invocation to run as an isolated subagent,
so that the orchestrator context stays clean and a failed agent doesn't crash the pipeline.

### Acceptance Criteria
- WHEN spawning an agent, THE SYSTEM SHALL use a subagent with its own isolated context window.
- WHEN a subagent fails mid-processing, THE SYSTEM SHALL log the error, skip the idea, and continue processing remaining ideas. The idea's status SHALL remain unchanged for retry on the next run.
- WHEN a subagent produces a malformed notes entry, THE SYSTEM SHALL log a warning and treat it as a no-op. The idea's status SHALL remain unchanged.
