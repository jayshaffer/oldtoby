# Decision Agent and Refinement Tasks

## 1. Decision agent command
- [ ] Create `.claude/commands/agent-decision.md` with decision agent prompt
- [ ] Implement context loading: read one-pager and notes.md fully; skim prd.md, market-analysis.md, solution-doc.md, financial-assessment.md (verdict summaries only)
- [ ] Implement verdict counting: tally pass/fail across all agent notes entries
- [ ] Implement concern aggregation: read all concerns from all agents, including passing agents
- [ ] Implement promote logic: set verdict `promoted`, next status `promoted`
- [ ] Implement kill logic: set verdict `killed`, next status `killed`
- [ ] Implement surgical send-back: identify weakest artifact, route to owning agent, write refinement directive with target agent, target artifact, what needs to change, what's working, and downstream impact
- [ ] Implement circuit breaker mode: when invoked with force flag, disallow send-back — must promote or kill
- [ ] Implement context bloat detection: if `bloat_kill_signal` enabled and idea folder exceeds ~8000 tokens, treat as negative signal
- [ ] Implement notes.md append with structured format

## 2. Send-back helper command
- [ ] Create `.claude/commands/send-back.md` accepting idea slug as `$ARGUMENTS`
- [ ] Implement folder move from `review/` to `pipeline/`
- [ ] Implement frontmatter update: set `status: brainstormed` in one-pager
- [ ] Implement feedback.md verification: warn if missing
- [ ] Implement idea-registry.md update for re-entered idea

## 3. Downstream cascade support
- [ ] Add refresh pass routing to orchestrator: after a send-back revision completes, check the refinement directive's `Downstream impact` for which agents need refresh
- [ ] Ensure refresh passes are dispatched only for named downstream agents
- [ ] Ensure refresh pass agents can escalate with `delegate-back:decision` if they disagree

## 4. Testing
- [ ] Test: Run `/agent-decision` on a cfo-ready idea with all artifacts, verify promote verdict
- [ ] Test: Run `/agent-decision` on an idea with weak market analysis, verify surgical send-back to market agent with refinement directive
- [ ] Test: Verify circuit breaker — run decision agent with iteration at max, confirm no send-back option
- [ ] Test: Run `/send-back` on an idea in review/, verify it moves to pipeline/ with status brainstormed
- [ ] Test: Verify downstream cascade — after PM revision from send-back, confirm market/tech/cfo get refresh passes
- [ ] Test: Verify human re-entry — PM picks up idea with feedback.md, revises PRD, downstream agents refresh
