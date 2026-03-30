# Review Agents Tasks

## 1. Agent profiles configuration
- [ ] Create `config/agent-profiles.md` with role definitions for all 5 agent roles (PM, Market, Tech, CFO, Decision)
- [ ] Define each role's: picks up status, promotes to status, mandate, evaluation criteria, artifact produced, delegation targets, failure threshold, and tone/personality

## 2. Product Engineer (PM) agent command
- [ ] Create `.claude/commands/agent-pm.md` with PM agent prompt
- [ ] Implement context loading: read one-pager fully, check for feedback.md, read notes.md fully
- [ ] Implement evaluation logic: coherence, specificity, differentiation checks
- [ ] Implement PRD generation with all required sections (Problem Definition, Target Users, User Stories/JTBD, MVP Feature Set, Success Metrics, Assumptions & Risks, Out of Scope, Open Questions)
- [ ] Implement re-entry handling for human feedback (revise PRD, don't restart; mark `Re-entry type: human-feedback`)
- [ ] Implement re-entry handling for decision agent send-back (revise specified gap only; mark `Re-entry type: decision-refinement`)
- [ ] Implement notes.md append with structured format (verdict, next status, iteration, assessment, concerns)

## 3. Market Fit Analyzer agent command
- [ ] Create `.claude/commands/agent-market.md` with market agent prompt
- [ ] Implement context loading: read one-pager and prd.md fully, skim feedback.md, read notes.md fully
- [ ] Implement evaluation logic: reachable market, plausible pricing, competitive moat
- [ ] Implement market-analysis.md generation with all required sections (TAM/SAM/SOM, Competitive Landscape, Acquisition Thesis, Pricing Validation, GTM Sketch, Risks, Verdict Summary)
- [ ] Implement delegation to PM when PRD is root cause of market concerns
- [ ] Implement notes.md append with structured format

## 4. Technical Lead agent command
- [ ] Create `.claude/commands/agent-tech.md` with tech lead prompt
- [ ] Implement context loading: read one-pager, prd.md, market-analysis.md, notes.md fully; skim feedback.md
- [ ] Implement evaluation logic: technical feasibility, build cost, infrastructure requirements
- [ ] Implement solution-doc.md generation with all required sections (Architecture, Tech Stack, Build vs Buy, MVP Scope/Effort, Infra Cost, Risks, Scalability, Timeline)
- [ ] Implement delegation to PM or Market when upstream artifacts are root cause
- [ ] Implement notes.md append with structured format

## 5. CFO agent command
- [ ] Create `.claude/commands/agent-cfo.md` with CFO agent prompt
- [ ] Implement context loading: read one-pager and notes.md fully, read solution-doc.md cost sections, skim prd.md and market-analysis.md pricing/revenue sections, skip solution-doc.md architecture
- [ ] Implement evaluation logic: unit economics, break-even, capital requirements with conservative assumptions
- [ ] Implement financial-assessment.md generation with all required sections (Unit Economics, Revenue Projections, Cost Structure, Break-Even, Capital Requirements, Financial Risks, ROI, Verdict Summary)
- [ ] Implement delegation to PM, Market, or Tech when upstream artifacts are root cause
- [ ] Implement notes.md append with structured format

## 6. Refresh pass behavior
- [ ] Add refresh pass instructions to each agent command: when re-processing after upstream change, read updated artifact + own prior work, confirm or revise verdict
- [ ] Ensure refresh pass notes entries are marked with `Pass type: refresh`
- [ ] Ensure refresh pass entries include `Changes from prior iteration` section

## 7. Testing
- [ ] Test: Run `/agent-pm` on a brainstormed idea, verify PRD is created and notes.md is appended correctly
- [ ] Test: Run `/agent-market` on a prd-ready idea, verify market-analysis.md is created
- [ ] Test: Run `/agent-tech` on a market-ready idea, verify solution-doc.md is created
- [ ] Test: Run `/agent-cfo` on a tech-ready idea, verify financial-assessment.md is created
- [ ] Test: Verify delegation — agent delegates back, notes contain delegation rationale and correct next status
- [ ] Test: Verify refresh pass — run agent after upstream change, confirm `Pass type: refresh` in notes
- [ ] Test: Verify human re-entry — run PM with feedback.md present, confirm PRD is revised (not restarted)
