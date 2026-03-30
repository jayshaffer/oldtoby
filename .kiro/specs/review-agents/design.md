# Review Agents Design

## Architecture

Each review agent is a Claude Code custom command in `.claude/commands/`. All four agents follow the same processing contract but have distinct mandates, reading strategies, and artifacts.

### Components
- `.claude/commands/agent-pm.md` — Product Engineer agent
- `.claude/commands/agent-market.md` — Market Fit Analyzer agent
- `.claude/commands/agent-tech.md` — Technical Lead agent
- `.claude/commands/agent-cfo.md` — CFO agent
- `config/agent-profiles.md` — Role definitions, mandates, evaluation criteria, and personality guidance

### Agent Command Template

Each command file follows this structure:

```markdown
# {Agent Role}

Read `config/agent-profiles.md` for your full role definition and evaluation criteria.
Read `config/constraints.md` for the current ideation constraints.

## Context Loading
{Per-agent reading instructions — which files to read fully, skim, or skip}

## Task
Process the idea at `$ARGUMENTS`.
{Role-specific evaluation instructions and artifact production guidance}

## Artifact Guidelines
{Target length, content expectations}

## Output
Append your assessment to notes.md using the structured notes format.
Your verdict line IS the output — the orchestrator reads it to determine next status.
```

## Data Flow

### Per-Agent Context Loading Strategy

| Agent | Read Fully | Skim (headers + verdict) | Skip |
|---|---|---|---|
| Product Engineer | one-pager, feedback.md, notes.md | — | — |
| Market Fit Analyzer | one-pager, prd.md, notes.md | feedback.md | — |
| Technical Lead | one-pager, prd.md, market-analysis.md, notes.md | feedback.md | — |
| CFO | one-pager, notes.md, solution-doc.md (cost sections) | prd.md, market-analysis.md (pricing/revenue only) | solution-doc.md (architecture) |

"Skim" means the command file instructs the agent to read only headers and verdict/summary sections.

### Notes Entry Format

```markdown
---
## [{Agent Role}] — {ISO timestamp}
**Verdict:** {pass | fail | delegate-back:{role} | refine}
**Next status:** {target status value}
**Iteration:** {n}
**Re-entry type:** {initial | human-feedback | decision-refinement | none}
**Pass type:** {full | refresh}

### Assessment
### Concerns
### Failure Rationale
### Delegation Rationale
### Changes from prior iteration
```

### Notes Growth Management

For contentious ideas with 5+ iterations, notes.md may have 10+ entries. Command files instruct agents to:
- Focus on the most recent entry from each agent role
- Focus on delegation/refinement directives
- Skim historical entries from the same agent (verdict line only)

### Artifact Formats

**PRD** (`prd.md`): Problem Definition, Target Users & Personas, User Stories / JTBD, Proposed Feature Set (MVP), Success Metrics, Assumptions & Risks, Out of Scope (v1), Open Questions for Market Fit

**Market Analysis** (`market-analysis.md`): TAM/SAM/SOM, Competitive Landscape, Customer Acquisition Thesis, Pricing Validation, Go-to-Market Sketch, Market Risks, Verdict Summary

**Solution Doc** (`solution-doc.md`): Architecture Overview, Tech Stack Recommendation, Build vs Buy, MVP Scope & Effort Estimate, Infra Cost Estimate (Monthly), Technical Risks & Unknowns, Scalability Considerations, Build Timeline (Rough)

**Financial Assessment** (`financial-assessment.md`): Unit Economics, Revenue Projections (12-month conservative), Cost Structure (Build + Run), Break-Even Analysis, Capital Requirements, Financial Risks, ROI Assessment, Verdict Summary

### Artifact Size Guidance

| Artifact | Target Length |
|---|---|
| PRD | 800-1200 words |
| Market analysis | 600-1000 words |
| Solution doc | 800-1200 words |
| Financial assessment | 500-800 words |
| Notes entry (per agent) | 200-400 words |

## Key Design Goals

1. **Adversarial by design**: Each agent has a distinct bias and pressure. The system kills bad ideas early, not rubber-stamps them.
2. **Read deeply what you own, skim what you don't**: Keeps each agent focused on its mandate without burning tokens on irrelevant context.
3. **Refinement over rejection**: Delegation sends ideas to the specific agent that can fix the issue, not back to square one.
4. **Stateless invocations**: Each agent reads the idea folder from scratch every time. No state carried between invocations.

## Agent Profile File

`config/agent-profiles.md` contains role definitions:

```markdown
## {Role Name}
**Picks up status:** {status value}
**Promotes to status:** {status value}
**Mandate:** {one-sentence mission}
**Evaluates for:** {bulleted criteria}
**Produces:** {artifact name}
**Can delegate to:** {list of roles}
**Failure threshold:** {what constitutes a kill}
**Tone:** {personality/perspective guidance}
```
