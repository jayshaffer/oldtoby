# Pipeline Orchestrator Tasks

## 1. Project scaffolding
- [x] Create directory structure: `config/`, `pipeline/`, `review/`, `trash/`, `archive/`, `logs/`, `logs/runs/`, `.claude/commands/`
- [x] Create `config/pipeline.config.json` with default configuration (schedule, brainstorm settings, agent thresholds, context limits, model config, logging flags)
- [x] Create empty `logs/changelog.md` with header
- [x] Create empty `logs/idea-registry.md` with header and table format
- [x] Create empty `logs/metrics.json` with initial zero counters

## 2. Orchestrator command file
- [ ] Create `.claude/commands/run-pipeline.md` with the orchestrator prompt
- [ ] Implement Phase 1: Run log idempotency check — read `logs/runs/` for today's date, conditionally invoke `/brainstorm` subagent
- [ ] Implement Phase 2: Scan `pipeline/` subfolders, read frontmatter `status` and `iteration` from each `one-pager.md`
- [ ] Implement Phase 2: Manual injection detection — check each idea against `logs/idea-registry.md`, append missing entries
- [ ] Implement Phase 2: Circuit breaker check — if `iteration >= max_iterations_per_idea`, route to decision agent with force flag
- [ ] Implement Phase 2: Stage-ordered dispatch — spawn appropriate agent subagent per status, process `brainstormed` first through `cfo-ready` last
- [ ] Implement Phase 2: Post-subagent verdict reading — read most recent `notes.md` entry, extract `**Verdict:**` and `**Next status:**`, update frontmatter
- [ ] Implement Phase 3: Reconciliation loop — re-scan for status changes, re-process up to `max_reconciliation_loops` times
- [ ] Implement Phase 4: Terminal moves — move `promoted` folders to `review/`, `killed` folders to `trash/`, update registry outcomes
- [ ] Implement Phase 5: Write pipeline snapshot to `logs/runs/{date}.log.md` with status counts, ideas per status, review/trash counts, iterations burned, reconciliation loops used, registry stats, thematic saturation warnings
- [ ] Implement Phase 5: Append to `logs/changelog.md` for promoted ideas using the changelog format (title, iterations, agents, human-refined flag, pitch, key concern, refinement history, build estimate, monthly cost)
- [ ] Implement Phase 5: Update `logs/metrics.json` with cumulative stats
- [ ] Implement Phase 5: Print `\a` terminal bell on promotion

## 3. Send-back helper command
- [ ] Create `.claude/commands/send-back.md` — accepts idea slug as `$ARGUMENTS`
- [ ] Implement: Move idea folder from `review/` to `pipeline/`, set `status: brainstormed` in frontmatter, preserve all existing artifacts
- [ ] Implement: Verify `feedback.md` exists in the idea folder (warn if missing)
- [ ] Implement: Append/update entry in `logs/idea-registry.md` for the re-entered idea

## 4. Integration testing
- [ ] Test: Create a mock idea folder in `pipeline/` with `status: brainstormed`, run orchestrator, verify PM agent is dispatched
- [ ] Test: Verify stage-ordered processing (earlier stages processed before later ones)
- [ ] Test: Verify idempotency — run twice on same date, confirm brainstorm is skipped on second run
- [ ] Test: Verify circuit breaker — set iteration to max, confirm decision agent is forced
- [ ] Test: Verify terminal moves — promoted idea lands in `review/`, killed idea lands in `trash/`
- [ ] Test: Verify manual injection — place idea without registry entry, confirm entry is created
- [ ] Test: Verify reconciliation — delegation changes status mid-run, confirm re-processing occurs
