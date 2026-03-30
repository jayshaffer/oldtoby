# Decision Agent and Refinement Tasks

## 1. Decision agent command
- [x] Create `.claude/commands/agent-decision.md` with decision agent prompt
- [x] Implement context loading: read one-pager and notes.md fully; skim prd.md, market-analysis.md, solution-doc.md, financial-assessment.md (verdict summaries only)
- [x] Implement verdict counting: tally pass/fail across all agent notes entries
- [x] Implement concern aggregation: read all concerns from all agents, including passing agents
- [x] Implement promote logic: set verdict `promoted`, next status `promoted`
- [x] Implement kill logic: set verdict `killed`, next status `killed`
- [x] Implement surgical send-back: identify weakest artifact, route to owning agent, write refinement directive with target agent, target artifact, what needs to change, what's working, and downstream impact
- [x] Implement circuit breaker mode: when invoked with force flag, disallow send-back — must promote or kill
- [x] Implement context bloat detection: if `bloat_kill_signal` enabled and idea folder exceeds ~8000 tokens, treat as negative signal
- [x] Implement notes.md append with structured format

## 2. Send-back helper command
- [x] Create `.claude/commands/send-back.md` accepting idea slug as `$ARGUMENTS`
- [x] Implement folder move from `review/` to `pipeline/`
- [x] Implement frontmatter update: set `status: brainstormed` in one-pager
- [x] Implement feedback.md verification: warn if missing
- [x] Implement idea-registry.md update for re-entered idea

## 3. Downstream cascade support
- [x] Add refresh pass routing to orchestrator: after a send-back revision completes, check the refinement directive's `Downstream impact` for which agents need refresh
- [x] Ensure refresh passes are dispatched only for named downstream agents
- [x] Ensure refresh pass agents can escalate with `delegate-back:decision` if they disagree

## 4. Testing
- [x] Test: Run `/agent-decision` on a cfo-ready idea with all artifacts, verify promote verdict
- [x] Test: Run `/agent-decision` on an idea with weak market analysis, verify surgical send-back to market agent with refinement directive
- [x] Test: Verify circuit breaker — run decision agent with iteration at max, confirm no send-back option
- [x] Test: Run `/send-back` on an idea in review/, verify it moves to pipeline/ with status brainstormed
- [x] Test: Verify downstream cascade — after PM revision from send-back, confirm market/tech/cfo get refresh passes
- [x] Test: Verify human re-entry — PM picks up idea with feedback.md, revises PRD, downstream agents refresh
