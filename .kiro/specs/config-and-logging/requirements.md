# Configuration and Logging Requirements

## User Story 1: Pipeline Configuration
As a product ideation operator, I want a centralized config file that controls all pipeline knobs,
so that I can tune behavior without editing command files.

### Acceptance Criteria
- WHEN the orchestrator starts, THE SYSTEM SHALL read `config/pipeline.config.json` for all configurable values.
- THE CONFIG SHALL include: schedule (cron expression, timezone), brainstorm settings (ideas per run, raw sketches count, dedup toggle, self-critique toggle), agent settings (max iterations, circuit breaker action, max reconciliation loops, refresh pass toggle), context limits (warn total tokens, bloat kill signal), model settings (brainstorm model, agent model), and logging settings (verbose, metrics, log brainstorm critique).

## User Story 2: Agent Profile Configuration
As a product ideation operator, I want agent profiles that define mandates and evaluation criteria without domain-specific heuristics,
so that I can shift ideation focus by updating constraints without rewriting agent profiles.

### Acceptance Criteria
- WHEN agents load their role, THE SYSTEM SHALL read `config/agent-profiles.md` for role definition, mandate, evaluation criteria, and personality guidance.
- THE AGENT PROFILES SHALL be generic — defining mandate and criteria, not domain-specific heuristics.
- THE CONSTRAINTS FILE (`config/constraints.md`) SHALL handle domain focus, keeping agents reusable across constraint configurations.

## User Story 3: Run Logging
As a product ideation operator, I want detailed per-run logs with pipeline snapshots,
so that I can monitor pipeline health and diagnose issues.

### Acceptance Criteria
- WHEN a run completes, THE SYSTEM SHALL write `logs/runs/{YYYY-MM-DD}.log.md` containing: pipeline snapshot (status counts, ideas per status), review/ and trash/ counts, total iterations burned, reconciliation loops used, registry stats (total, promoted, killed, duplicate, active), and thematic saturation warnings.
- WHEN verbose logging is enabled, THE SYSTEM SHALL include brainstorm critique reasoning, dedup decisions, and per-agent processing details in the run log.

## User Story 4: Changelog
As a product ideation operator, I want a scannable changelog of promoted ideas,
so that I can quickly see what the pipeline has produced without digging through folders.

### Acceptance Criteria
- WHEN an idea is promoted, THE SYSTEM SHALL append an entry to `logs/changelog.md`.
- WHEN writing a changelog entry, THE SYSTEM SHALL include: idea title (linked to review folder), date, iterations, agents passed, human-refined flag, elevator pitch, key concern, refinement history, estimated build timeline, and estimated monthly cost.

## User Story 5: Idea Registry
As the pipeline, I want an append-only registry of every idea ever generated,
so that the brainstorm agent has institutional memory and operators can browse full idea history.

### Acceptance Criteria
- WHEN any idea enters the pipeline (brainstormed, manually injected, or re-entered), THE SYSTEM SHALL have an entry in `logs/idea-registry.md`.
- WHEN an idea reaches a terminal state, THE SYSTEM SHALL update its registry entry with the final outcome.
- THE REGISTRY SHALL capture ideas across all outcomes: promoted, killed, trashed, archived, and duplicate.
- THE REGISTRY SHALL NOT be pruned — it is append-only institutional memory.

## User Story 6: Metrics
As a product ideation operator, I want cumulative pipeline metrics,
so that I can track pipeline effectiveness over time.

### Acceptance Criteria
- WHEN a run completes, THE SYSTEM SHALL update `logs/metrics.json` with cumulative stats: total ideas generated, total promoted, total killed, total duplicates, pass rates per agent, average iterations to promotion, average iterations to kill.

## User Story 7: Error Logging
As a product ideation operator, I want errors captured in run logs rather than silently swallowed,
so that I can diagnose pipeline issues after the fact.

### Acceptance Criteria
- WHEN a subagent fails, THE SYSTEM SHALL log the error in the run log with the idea slug and error details.
- WHEN constraints file is missing, THE SYSTEM SHALL log the error and continue processing existing ideas.
- WHEN a frontmatter status is unrecognized, THE SYSTEM SHALL log the error and skip the idea.
- WHEN the orchestrator hits a Claude Code usage limit mid-run, THE SYSTEM SHALL capture partial progress in the run log.
