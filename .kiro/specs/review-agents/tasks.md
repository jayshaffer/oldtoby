# Review Agents Tasks

## 1. Agent profiles configuration
- [x] Create `config/agent-profiles.md` with role definitions for all 5 agent roles (PM, Market, Tech, CFO, Decision)
- [x] Define each role's: picks up status, promotes to status, mandate, evaluation criteria, artifact produced, delegation targets, failure threshold, and tone/personality

## 2. Product Engineer (PM) agent command
- [x] Create `.claude/commands/agent-pm.md` with PM agent prompt
- [x] Implement context loading: read one-pager fully, check for feedback.md, read notes.md fully
- [x] Implement evaluation logic: coherence, specificity, differentiation checks
- [x] Implement PRD generation with all required sections (Problem Definition, Target Users, User Stories/JTBD, MVP Feature Set, Success Metrics, Assumptions & Risks, Out of Scope, Open Questions)
- [x] Implement re-entry handling for human feedback (revise PRD, don't restart; mark `Re-entry type: human-feedback`)
- [x] Implement re-entry handling for decision agent send-back (revise specified gap only; mark `Re-entry type: decision-refinement`)
- [x] Implement notes.md append with structured format (verdict, next status, iteration, assessment, concerns)

## 3. Market Fit Analyzer agent command
- [x] Create `.claude/commands/agent-market.md` with market agent prompt
- [x] Implement context loading: read one-pager and prd.md fully, skim feedback.md, read notes.md fully
- [x] Implement evaluation logic: reachable market, plausible pricing, competitive moat
- [x] Implement market-analysis.md generation with all required sections (TAM/SAM/SOM, Competitive Landscape, Acquisition Thesis, Pricing Validation, GTM Sketch, Risks, Verdict Summary)
- [x] Implement delegation to PM when PRD is root cause of market concerns
- [x] Implement notes.md append with structured format

## 4. Technical Lead agent command
- [x] Create `.claude/commands/agent-tech.md` with tech lead prompt
- [x] Implement context loading: read one-pager, prd.md, market-analysis.md, notes.md fully; skim feedback.md
- [x] Implement evaluation logic: technical feasibility, build cost, infrastructure requirements
- [x] Implement solution-doc.md generation with all required sections (Architecture, Tech Stack, Build vs Buy, MVP Scope/Effort, Infra Cost, Risks, Scalability, Timeline)
- [x] Implement delegation to PM or Market when upstream artifacts are root cause
- [x] Implement notes.md append with structured format

## 5. CFO agent command
- [x] Create `.claude/commands/agent-cfo.md` with CFO agent prompt
- [x] Implement context loading: read one-pager and notes.md fully, read solution-doc.md cost sections, skim prd.md and market-analysis.md pricing/revenue sections, skip solution-doc.md architecture
- [x] Implement evaluation logic: unit economics, break-even, capital requirements with conservative assumptions
- [x] Implement financial-assessment.md generation with all required sections (Unit Economics, Revenue Projections, Cost Structure, Break-Even, Capital Requirements, Financial Risks, ROI, Verdict Summary)
- [x] Implement delegation to PM, Market, or Tech when upstream artifacts are root cause
- [x] Implement notes.md append with structured format

## 6. Refresh pass behavior
- [x] Add refresh pass instructions to each agent command: when re-processing after upstream change, read updated artifact + own prior work, confirm or revise verdict
- [x] Ensure refresh pass notes entries are marked with `Pass type: refresh`
- [x] Ensure refresh pass entries include `Changes from prior iteration` section

## 7. Testing
- [x] Test: Run `/agent-pm` on a brainstormed idea, verify PRD is created and notes.md is appended correctly
- [x] Test: Run `/agent-market` on a prd-ready idea, verify market-analysis.md is created
- [x] Test: Run `/agent-tech` on a market-ready idea, verify solution-doc.md is created
- [x] Test: Run `/agent-cfo` on a tech-ready idea, verify financial-assessment.md is created
- [x] Test: Verify delegation — agent delegates back, notes contain delegation rationale and correct next status
- [x] Test: Verify refresh pass — run agent after upstream change, confirm `Pass type: refresh` in notes
- [x] Test: Verify human re-entry — run PM with feedback.md present, confirm PRD is revised (not restarted)
