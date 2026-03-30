# Review Agents Requirements

## User Story 1: Product Engineer (PM) Agent
As the pipeline, I want a PM agent that transforms raw ideas into structured PRDs,
so that downstream agents have a well-defined product scope to evaluate.

### Acceptance Criteria
- WHEN an idea has `status: brainstormed`, THE SYSTEM SHALL invoke `/agent-pm` with the idea folder path.
- WHEN processing, THE SYSTEM SHALL read `one-pager.md` fully, check for `feedback.md` (human re-entry), and read `notes.md` fully.
- WHEN the idea is coherent, specific, and non-trivially differentiated, THE SYSTEM SHALL write `prd.md` and set verdict to `pass` with `Next status: prd-ready`.
- WHEN the idea is incoherent or trivially duplicative, THE SYSTEM SHALL set verdict to `fail` with `Next status: killed`.
- WHEN `feedback.md` exists (human re-entry), THE SYSTEM SHALL revise the existing PRD to address feedback rather than starting from scratch, and mark notes with `Re-entry type: human-feedback`.
- WHEN re-entering via decision agent send-back, THE SYSTEM SHALL revise only the specified artifact gap and mark notes with `Re-entry type: decision-refinement`.
- THE PM AGENT SHALL NOT delegate backward (it is the first agent in the chain).

## User Story 2: Market Fit Analyzer Agent
As the pipeline, I want a market fit agent that pressure-tests whether a viable market exists,
so that technically feasible but commercially unviable ideas are filtered out.

### Acceptance Criteria
- WHEN an idea has `status: prd-ready`, THE SYSTEM SHALL invoke `/agent-market` with the idea folder path.
- WHEN processing, THE SYSTEM SHALL read `one-pager.md` and `prd.md` fully, skim `feedback.md` (headers + verdict only), and read `notes.md` fully.
- WHEN a reachable market exists with plausible pricing and a real competitive moat, THE SYSTEM SHALL write `market-analysis.md` and set verdict to `pass` with `Next status: market-ready`.
- WHEN the market is unreachable, pricing is implausible, or there is no moat, THE SYSTEM SHALL set verdict to `fail` with `Next status: killed`.
- WHEN the PRD is the root cause of market concerns, THE SYSTEM SHALL set verdict to `delegate-back:pm` with `Next status: brainstormed`.

## User Story 3: Technical Lead Agent
As the pipeline, I want a tech lead agent that evaluates technical viability and estimates build cost,
so that ideas with unrealistic technical requirements are caught before financial review.

### Acceptance Criteria
- WHEN an idea has `status: market-ready`, THE SYSTEM SHALL invoke `/agent-tech` with the idea folder path.
- WHEN processing, THE SYSTEM SHALL read `one-pager.md`, `prd.md`, `market-analysis.md`, and `notes.md` fully, and skim `feedback.md`.
- WHEN the idea is technically viable, THE SYSTEM SHALL write `solution-doc.md` (architecture overview, tech stack, build vs buy, MVP scope, effort estimate, infra cost, risks, scalability, timeline) and set verdict to `pass` with `Next status: tech-ready`.
- WHEN the idea is technically infeasible or cost-prohibitive to build, THE SYSTEM SHALL set verdict to `fail` with `Next status: killed`.
- WHEN upstream artifacts (PRD or market analysis) are the root cause, THE SYSTEM SHALL set verdict to `delegate-back:{pm|market}` with appropriate `Next status`.

## User Story 4: CFO Agent
As the pipeline, I want a CFO agent that evaluates financial viability with conservative assumptions,
so that ideas with broken unit economics don't waste human review time.

### Acceptance Criteria
- WHEN an idea has `status: tech-ready`, THE SYSTEM SHALL invoke `/agent-cfo` with the idea folder path.
- WHEN processing, THE SYSTEM SHALL read `one-pager.md`, `notes.md`, and `solution-doc.md` cost sections fully; skim `prd.md` and `market-analysis.md` pricing/revenue sections only; skip `solution-doc.md` architecture sections.
- WHEN the numbers work with conservative assumptions, THE SYSTEM SHALL write `financial-assessment.md` (unit economics, 12-month revenue projections, cost structure, break-even, capital requirements, financial risks, ROI) and set verdict to `pass` with `Next status: cfo-ready`.
- WHEN the unit economics are broken or capital requirements are unrealistic, THE SYSTEM SHALL set verdict to `fail` with `Next status: killed`.
- WHEN upstream artifacts are the root cause, THE SYSTEM SHALL set verdict to `delegate-back:{pm|market|tech}` with appropriate `Next status`.

## User Story 5: Agent Processing Contract
As the pipeline, I want all review agents to follow a consistent processing contract,
so that the orchestrator can reliably read verdicts and route ideas.

### Acceptance Criteria
- WHEN any agent completes processing, THE SYSTEM SHALL append a structured entry to `notes.md` with: Agent Role, ISO timestamp, Verdict, Next status, Iteration, Re-entry type, Pass type.
- WHEN verdict is `pass`, THE SYSTEM SHALL include an Assessment section and a Concerns section (required).
- WHEN verdict is `fail`, THE SYSTEM SHALL include a Failure Rationale section (required).
- WHEN verdict is `delegate-back`, THE SYSTEM SHALL include a Delegation Rationale section (required).
- WHEN processing a refresh pass (upstream artifact changed), THE SYSTEM SHALL read the updated artifact and their own prior work, confirm or revise their verdict, and mark with `Pass type: refresh`.
- WHEN producing artifacts, THE SYSTEM SHALL respect target length guidelines: PRD 800-1200 words, market analysis 600-1000 words, solution doc 800-1200 words, financial assessment 500-800 words, notes entry 200-400 words.

## User Story 6: Agent Personality and Bias
As the pipeline, I want each agent to have a distinct adversarial bias,
so that the review gauntlet kills bad ideas rather than rubber-stamping them.

### Acceptance Criteria
- WHEN the PM agent evaluates, THE SYSTEM SHALL apply builder optimism tempered by scope awareness — "Is this specific enough to build? Is the scope realistic for MVP?"
- WHEN the market agent evaluates, THE SYSTEM SHALL apply skeptical realism — "Show me the customer. Show me they'll pay. Show me the moat."
- WHEN the tech lead evaluates, THE SYSTEM SHALL apply pragmatic engineering — "Can we actually build this? What's the real cost? What breaks at scale?"
- WHEN the CFO evaluates, THE SYSTEM SHALL apply conservative financial lens — "Do the numbers work with conservative assumptions? What's the burn?"
