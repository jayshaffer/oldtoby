# Configuration and Logging Tasks

## 1. Configuration files
- [x] Create `config/pipeline.config.json` with all default values (schedule, brainstorm, agents, context, model, logging sections)
- [x] Create `config/agent-profiles.md` with role definitions for PM, Market, Tech, CFO, and Decision agents (picks up status, promotes to status, mandate, evaluates for, produces, can delegate to, failure threshold, tone)
- [x] Create `config/constraints.md` with template sections (Domain Focus, Technical Boundaries, Market Preferences, Anti-Patterns, Current Interests) and placeholder guidance comments

## 2. Log file initialization
- [x] Create `logs/changelog.md` with header
- [x] Create `logs/idea-registry.md` with header and table format (Slug, Title, Problem, Keywords, Status, Date, Outcome)
- [x] Create `logs/metrics.json` with initial zero counters for all metrics
- [x] Create `logs/runs/` directory

## 3. Logging implementation in orchestrator
- [x] Implement run log writing: pipeline snapshot with status counts, ideas per status, review/trash counts, iterations burned, reconciliation loops used, registry stats, thematic saturation
- [x] Implement verbose logging: brainstorm critique reasoning, dedup decisions, per-agent processing details (when `logging.verbose` is true)
- [x] Implement changelog append: title linked to review folder, date, iterations, agents passed, human-refined flag, pitch, key concern, refinement history, build estimate, monthly cost
- [x] Implement registry updates: append new entries, update outcomes for terminal ideas
- [x] Implement metrics update: increment counters, recalculate pass rates and averages

## 4. Error logging
- [x] Implement subagent failure logging with idea slug and error details
- [x] Implement missing constraints file logging (abort brainstorm, continue processing)
- [x] Implement unrecognized status logging (skip idea)
- [x] Implement partial progress capture when usage limit is hit mid-run

## 5. Testing
- [x] Test: Verify pipeline.config.json is valid JSON and all expected keys are present
- [x] Test: Verify agent-profiles.md contains entries for all 5 agent roles
- [x] Test: Verify run log includes correct pipeline snapshot format
- [x] Test: Verify changelog entry includes all required fields
- [x] Test: Verify registry is append-only — new entries don't overwrite existing ones
- [x] Test: Verify metrics.json updates correctly after a run with promotions and kills
