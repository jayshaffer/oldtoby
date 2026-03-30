# Overall Task List

This is the composed master task list for the pipeline orchestrator system. Each section delegates to its respective spec's task file.

## Phase 1: Project Scaffolding & Configuration
> See [config-and-logging/tasks.md](config-and-logging/tasks.md) — Tasks 1–3

- [x] Create directory structure and all configuration files (`pipeline.config.json`, `agent-profiles.md`, `constraints.md`)
- [x] Initialize all log files (`changelog.md`, `idea-registry.md`, `metrics.json`, `logs/runs/`)

## Phase 2: Brainstorm Agent
> See [brainstorm-agent/tasks.md](brainstorm-agent/tasks.md) — Tasks 1–4

- [x] Create constraints file template
- [x] Create `.claude/commands/brainstorm.md` with full prompt (context loading, generation, dedup, self-critique, selection, write, registry update)
- [ ] Validate one-pager template (frontmatter, sections, word count)
- [ ] Test brainstorm agent (empty registry, dedup, saturation, format)

## Phase 3: Review Agents
> See [review-agents/tasks.md](review-agents/tasks.md) — Tasks 1–7

- [x] Create `config/agent-profiles.md` with all 5 role definitions
- [ ] Create `.claude/commands/agent-pm.md` (PM agent — PRD generation, re-entry handling)
- [ ] Create `.claude/commands/agent-market.md` (Market agent — market-analysis.md, delegation)
- [ ] Create `.claude/commands/agent-tech.md` (Tech agent — solution-doc.md, delegation)
- [ ] Create `.claude/commands/agent-cfo.md` (CFO agent — financial-assessment.md, delegation)
- [ ] Add refresh pass behavior to all agents
- [ ] Test review agents (artifact creation, delegation, refresh, re-entry)

## Phase 4: Decision Agent & Refinement
> See [decision-and-refinement/tasks.md](decision-and-refinement/tasks.md) — Tasks 1–4

- [ ] Create `.claude/commands/agent-decision.md` (verdict counting, promote/kill/send-back, circuit breaker, bloat detection)
- [ ] Create `.claude/commands/send-back.md` (move from review to pipeline, registry update)
- [ ] Implement downstream cascade support (refresh pass routing, escalation)
- [ ] Test decision agent (promote, send-back, circuit breaker, cascade, re-entry)

## Phase 5: Pipeline Orchestrator
> See [pipeline-orchestrator/tasks.md](pipeline-orchestrator/tasks.md) — Tasks 2–4

- [ ] Create `.claude/commands/run-pipeline.md` with orchestrator prompt
- [ ] Implement Phase 1: Idempotency check + brainstorm invocation
- [ ] Implement Phase 2: Pipeline scan, manual injection, circuit breaker, stage-ordered dispatch, verdict reading
- [ ] Implement Phase 3: Reconciliation loop
- [ ] Implement Phase 4: Terminal moves (promoted → `review/`, killed → `trash/`)
- [ ] Implement Phase 5: Run log, changelog, metrics, terminal bell
- [ ] Create send-back helper command
- [ ] Test orchestrator (dispatch, ordering, idempotency, circuit breaker, terminal moves, injection, reconciliation)

## Phase 6: Logging Implementation
> See [config-and-logging/tasks.md](config-and-logging/tasks.md) — Tasks 3–5

- [ ] Implement run log writing and verbose logging
- [ ] Implement changelog append and registry updates
- [ ] Implement metrics updates
- [ ] Implement error logging (subagent failure, missing constraints, unrecognized status, partial progress)
- [ ] Test logging (snapshot format, changelog fields, append-only registry, metrics correctness)
