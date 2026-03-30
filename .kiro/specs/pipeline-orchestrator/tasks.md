# Pipeline Orchestrator Tasks

## 1. Project scaffolding
- [x] Create directory structure: `config/`, `pipeline/`, `review/`, `trash/`, `archive/`, `logs/`, `logs/runs/`, `.claude/commands/`
- [x] Create `config/pipeline.config.json` with default configuration (schedule, brainstorm settings, agent thresholds, context limits, model config, logging flags)
- [x] Create empty `logs/changelog.md` with header
- [x] Create empty `logs/idea-registry.md` with header and table format
- [x] Create empty `logs/metrics.json` with initial zero counters

## 2. Orchestrator command file
- [x] Create `.claude/commands/run-pipeline.md` with the orchestrator prompt
- [x] Implement Phase 1: Run log idempotency check — read `logs/runs/` for today's date, conditionally invoke `/brainstorm` subagent
- [x] Implement Phase 2: Scan `pipeline/` subfolders, read frontmatter `status` and `iteration` from each `one-pager.md`
- [x] Implement Phase 2: Manual injection detection — check each idea against `logs/idea-registry.md`, append missing entries
- [x] Implement Phase 2: Circuit breaker check — if `iteration >= max_iterations_per_idea`, route to decision agent with force flag
- [x] Implement Phase 2: Stage-ordered dispatch — spawn appropriate agent subagent per status, process `brainstormed` first through `cfo-ready` last
- [x] Implement Phase 2: Post-subagent verdict reading — read most recent `notes.md` entry, extract `**Verdict:**` and `**Next status:**`, update frontmatter
- [x] Implement Phase 3: Reconciliation loop — re-scan for status changes, re-process up to `max_reconciliation_loops` times
- [x] Implement Phase 4: Terminal moves — move `promoted` folders to `review/`, `killed` folders to `trash/`, update registry outcomes
- [x] Implement Phase 5: Write pipeline snapshot to `logs/runs/{date}.log.md` with status counts, ideas per status, review/trash counts, iterations burned, reconciliation loops used, registry stats, thematic saturation warnings
- [x] Implement Phase 5: Append to `logs/changelog.md` for promoted ideas using the changelog format (title, iterations, agents, human-refined flag, pitch, key concern, refinement history, build estimate, monthly cost)
- [x] Implement Phase 5: Update `logs/metrics.json` with cumulative stats
- [x] Implement Phase 5: Print `\a` terminal bell on promotion

## 3. Send-back helper command
- [x] Create `.claude/commands/send-back.md` — accepts idea slug as `$ARGUMENTS`
- [x] Implement: Move idea folder from `review/` to `pipeline/`, set `status: brainstormed` in frontmatter, preserve all existing artifacts
- [x] Implement: Verify `feedback.md` exists in the idea folder (warn if missing)
- [x] Implement: Append/update entry in `logs/idea-registry.md` for the re-entered idea

## 4. Integration testing
- [x] Test: Structural audit — cross-referenced all agent contracts, status values, verdict formats, frontmatter fields, log formats, config keys, and registry format against the orchestrator
- [x] Test: Verified stage-ordered processing logic in dispatch step
- [x] Test: Verified idempotency — run log existence check gates brainstorm phase
- [x] Test: Verified circuit breaker — iteration check routes to decision agent
- [x] Test: Verified terminal moves — promoted/killed folders moved with append-only registry rows
- [x] Test: Verified manual injection — missing registry entry triggers append
- [x] Test: Verified reconciliation — loop respects max_reconciliation_loops and refresh_pass config flag
