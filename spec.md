# Product Ideation Pipeline — Specification

**Version:** 0.8.0-draft
**Author:** Jay + Claude
**Date:** 2026-03-28

---

## 1. Overview

An autonomous nightly pipeline powered by Claude Code that brainstorms software product/service ideas, then runs each idea through a multi-agent review gauntlet before promoting survivors to a human review folder. The system uses a folder-based state machine on the local filesystem — no external services required beyond Claude Code itself.

### Design Principles

- **Filesystem as state.** Three folders: `pipeline/` (active work), `review/` (human output), `trash/` (dead ideas). An idea's stage is tracked via frontmatter `status` in its one-pager — the orchestrator command reads status and dispatches to the right agent. No database, no API, no shell glue — just Claude Code commands and files.
- **Nightly batch processor, not a synchronous pipeline.** Each run advances every idea by one step. An idea brainstormed Monday might not promote until Wednesday. Ideas at different stages coexist in `pipeline/`, each progressing at its own pace. Runs are cheap and incremental.
- **Constraints-driven ideation.** Every brainstorm session reads a human-authored constraints file so the pipeline stays focused on ideas the operator actually cares about.
- **Adversarial review, not consensus.** Each agent role has a distinct mandate and incentive. The system is designed to kill bad ideas early, not rubber-stamp them.
- **Human-in-the-loop at the exit.** Nothing ships without landing in the review folder where the operator reads it. The pipeline is a filter, not a decision-maker.
- **Refinement over rejection.** The pipeline prefers surgical refinement to wholesale restarts. Self-critique catches low-quality output before it enters the pipeline. Post-decision routing sends ideas back to the specific agent that can fix the issue. Human feedback re-enters the pipeline as a first-class input, not an out-of-band override.

---

## 2. Directory Structure

```
ideation-pipeline/
├── config/
│   ├── constraints.md            # Human-authored ideation constraints
│   ├── agent-profiles.md         # Role definitions, mandates, evaluation criteria
│   └── pipeline.config.json      # Schedule, thresholds, knobs
│
├── pipeline/                     # All active ideas, at any stage
│   └── {idea-slug}/
│       ├── one-pager.md          # Frontmatter `status` drives routing
│       ├── notes.md              # Append-only agent commentary
│       ├── feedback.md           # Human feedback (only on re-entry from review/)
│       ├── prd.md                # Created by PM agent
│       ├── market-analysis.md    # Created by Market Fit agent
│       ├── solution-doc.md       # Created by Tech Lead agent
│       └── financial-assessment.md # Created by CFO agent
│
├── review/                       # Human review — pipeline output
│   └── {idea-slug}/
│       └── ... (full artifact bundle)
│
├── trash/                        # Rejected ideas with kill rationale
│   └── {idea-slug}/
│       └── ...
│
├── archive/                      # Previously reviewed (moved by human)
│   └── {idea-slug}/
│       └── ...
│
├── logs/
│   ├── changelog.md              # Append-only log of promoted ideas
│   ├── idea-registry.md          # Master index of every idea ever generated
│   ├── runs/
│   │   └── {YYYY-MM-DD}.log.md  # Per-run execution log (includes pipeline snapshot)
│   └── metrics.json              # Cumulative stats (ideas generated, pass rates, etc.)
│
├── .claude/
│   └── commands/
│       ├── run-pipeline.md       # Orchestrator: dispatch loop, frontmatter, logging
│       ├── send-back.md          # Helper: move idea from review/ to pipeline/ with feedback
│       ├── brainstorm.md         # Agent: brainstorm phase
│       ├── agent-pm.md           # Agent: product engineer
│       ├── agent-market.md       # Agent: market fit analyzer
│       ├── agent-tech.md         # Agent: technical lead
│       ├── agent-cfo.md          # Agent: CFO
│       └── agent-decision.md     # Agent: final decision agent
```

Artifacts accumulate in the idea folder as agents process it. Only the files that exist at any given point are the ones that have been created — a freshly brainstormed idea only has `one-pager.md` and `notes.md`. By the time it reaches the decision agent, the full bundle is present.

---

## 3. Pipeline Stages

### 3.1 Trigger & Schedule

| Property | Value |
|---|---|
| Schedule | Nightly, outside peak hours (configurable, default `02:00 Mountain`) |
| Trigger | Cron invokes `claude -p "/run-pipeline" --model sonnet` |
| Idempotency | Each run is date-stamped. If a run log exists for today, skip brainstorm but still process any ideas sitting at intermediate statuses. |

The `/run-pipeline` command is the single entry point. It handles everything:
1. Check for existing run log → skip brainstorm if already ran today.
2. Invoke `/brainstorm` → writes new ideas to `pipeline/` with `status: brainstormed`.
3. Scan `pipeline/` for all idea folders. Read `status` from each one-pager's frontmatter. Invoke the appropriate agent command for each idea based on status.
4. Update frontmatter `status` and `iteration` based on each agent's verdict.
5. Run reconciliation loops for delegated/sent-back ideas (up to N per run).
6. Move terminal ideas: `status: promoted` → `review/`, `status: killed` → `trash/`.
7. Append to `logs/changelog.md` for newly promoted ideas.
8. Write pipeline snapshot and run summary to `logs/runs/{date}.log.md`.

Cron entry:

```
0 2 * * * cd /path/to/ideation-pipeline && claude -p "/run-pipeline" --model sonnet
```

### 3.2 Brainstorm Phase

**Input:** `config/constraints.md`, `logs/idea-registry.md`
**Output:** 5 one-pager files in `pipeline/` with `status: brainstormed`, plus registry entries for all generated ideas (including duplicates)

The brainstorm agent reads the constraints file and the idea registry, then generates product/service ideas through a generate → dedup → critique process.

#### Self-Critique Process

The brainstorm agent does NOT generate 5 ideas and dump them. Instead:

1. **Load context.** Read `config/constraints.md` and `logs/idea-registry.md`.
2. **Generate.** Produce 8–10 raw idea sketches (working title + 2-sentence pitch + target customer + keywords). These are internal working notes, never written to disk.
3. **Dedup.** Check each sketch against the idea registry (see Deduplication below). Flag and discard duplicates. Generate replacement sketches for each duplicate, up to 3 retries per slot.
4. **Critique.** Re-read surviving sketches against the constraints file and evaluate on two axes:
   - **Constraint alignment** — does it fit the domain focus, technical boundaries, and market preferences?
   - **Specificity** — is the problem concrete enough that a PM could write a PRD, or is it a vague category ("AI for healthcare")?
5. **Select.** Rank the sketches and select the top 5. For each selected idea, tighten the pitch based on critique findings before writing the one-pager.
6. **Write.** Produce the 5 one-pager files in `pipeline/`. Append all ideas (selected and duplicates) to `logs/idea-registry.md`.

The entire brainstorm step is a single Claude Code invocation — generate, dedup, and critique happen in the same context window. The raw sketches, dedup decisions, and critique reasoning are logged to the run log but not persisted as pipeline artifacts.

#### Constraints File Format (`config/constraints.md`)

Human-authored. Freeform markdown with the following suggested sections:

```markdown
# Ideation Constraints

## Domain Focus
<!-- What spaces are you interested in? SaaS, dev tools, healthcare, etc. -->

## Technical Boundaries
<!-- What tech are you willing to build with? What's off-limits? -->

## Market Preferences
<!-- B2B vs B2C, target revenue range, market size floor, etc. -->

## Anti-Patterns
<!-- Ideas to explicitly avoid. "No crypto." "No social media." etc. -->

## Current Interests
<!-- What's top of mind? Themes, problems you've noticed, trends. -->
```

#### One-Pager Format (`one-pager.md`)

```markdown
# {Idea Title}

**Slug:** {kebab-case-slug}
**Generated:** {ISO timestamp}
**Status:** brainstormed
**Iteration:** 0
**Keywords:** {comma-separated list of domain terms, problem-space words, solution-shape words}

## Elevator Pitch
<!-- 2-3 sentences. What is it? Who is it for? Why now? -->

## Problem Statement
<!-- What pain point does this address? Who feels it? -->

## Proposed Solution
<!-- High-level approach. No implementation details. -->

## Target Customer
<!-- Persona sketch. Role, company size, budget authority. -->

## Revenue Model
<!-- How does this make money? Pricing shape. -->

## Differentiation
<!-- Why wouldn't someone just use {existing thing}? -->

## Open Questions
<!-- What would need to be true for this to work? -->
```

#### Deduplication

Slug matching alone is insufficient — "api-health-monitor" and "endpoint-uptime-tracker" are different slugs for the same idea. The pipeline maintains an **idea registry** that enables semantic dedup across the full history.

**Idea Registry** (`logs/idea-registry.md`)

Append-only. Every idea that enters the pipeline gets a one-line entry, regardless of outcome. The brainstorm agent, the orchestrator, and the `/send-back` command all append to this file when new ideas are created or re-enter the pipeline.

```markdown
# Idea Registry

| Slug | Title | Problem (one-line) | Keywords | Status | Date | Outcome |
|---|---|---|---|---|---|---|
| api-health-monitor | API Health Monitor | Devs need uptime alerts without configuring Datadog | api, monitoring, uptime, alerts, devs | killed | 2026-03-28 | Killed at market-fit: no moat |
| endpoint-uptime-tracker | Endpoint Uptime Tracker | SaaS teams need simple endpoint monitoring | api, monitoring, uptime, saas | duplicate | 2026-03-29 | Deduped against api-health-monitor |
| invoice-autopilot | Invoice Autopilot | Freelancers waste hours on invoicing | invoicing, freelancer, automation, billing | promoted | 2026-03-28 | Promoted to review |
```

The `Keywords` column is critical — it's a flat list of domain terms, problem-space words, and solution-shape words that the brainstorm agent generates alongside each idea. These are what make semantic matching possible without a vector database.

**Dedup Process (during brainstorm)**

After the brainstorm agent generates its 8–10 raw sketches and before the self-critique step:

1. **Read the registry.** Load `logs/idea-registry.md` into context.
2. **For each sketch, check against registry entries.** Compare the sketch's problem statement, keywords, and solution shape against every registry entry. Two ideas are duplicates if they solve the same core problem for the same customer, even if the proposed solution differs. "API health monitor" and "endpoint uptime tracker" are the same idea. "API health monitor" and "API documentation generator" are not.
3. **Flag duplicates.** If a sketch matches an existing entry, mark it as a duplicate and note which registry entry it matched. Do not write it to `pipeline/`.
4. **Log skipped duplicates.** Append duplicate entries to the registry with `Status: duplicate` and `Outcome: Deduped against {matched-slug}`. This prevents the same idea from being re-generated and re-checked on future runs.
5. **Generate replacements.** For each duplicate flagged, generate a replacement sketch. Up to 3 replacement attempts per slot before accepting whatever is generated.

**Dedup Process (during human injection)**

When the operator creates a new idea folder in `pipeline/` or when `/send-back` re-introduces an idea:

1. The orchestrator checks the slug against the registry during the next `/run-pipeline`.
2. If the idea already has a registry entry (re-entry via send-back), no action needed — it's a known idea being refined.
3. If the slug is new (manual injection), the orchestrator adds a registry entry before dispatching to the PM agent.

**Dedup Scope**

The registry captures ideas across all outcomes — promoted, killed, trashed, archived, and duplicate. This means the brainstorm agent won't regenerate:
- Ideas that were killed (they failed for a reason).
- Ideas that were promoted (they're already in review or archived).
- Ideas that were previously flagged as duplicates.

**Registry Maintenance**

The registry is append-only and should not be pruned. Over time it becomes the brainstorm agent's institutional memory. If the operator wants the brainstorm agent to revisit a previously killed idea space (e.g., "monitoring" ideas that were killed 6 months ago might be viable now), they can:
- Add a note to `config/constraints.md` explicitly requesting ideas in that space.
- The brainstorm agent sees both the constraint and the registry entries, and can generate a meaningfully differentiated take on the same problem space.

**Thematic drift detection**

If the brainstorm agent notices that 3+ of its 8–10 raw sketches match existing registry entries, it should flag this in the run log as a **thematic saturation warning**. This means the constraints file is too narrow or the idea space is exhausted — the operator should update constraints to open new territory.

### 3.3 Agent Processing Model

All four review agents follow the same processing contract:

#### Input
- The idea folder in `pipeline/` (dispatched by the `/run-pipeline` orchestrator based on frontmatter `status`).
- Files in the folder, read selectively per the context management strategy (see below).
- `config/agent-profiles.md` for role context and evaluation criteria.
- Awareness of which other agents exist and their mandates (for delegation context).

#### Output (per idea)
1. **Stage artifact** — the document this agent is responsible for creating (PRD, market analysis, solution doc, financial assessment).
2. **Notes entry** — appended to `notes.md` in a structured format (see below). The verdict metadata in the notes entry IS the output — the orchestrator reads the most recent entry's `**Verdict:**` and `**Next status:**` lines to determine routing.

The `/run-pipeline` orchestrator reads the most recent notes entry, extracts the verdict, updates the `status` and `iteration` fields in the one-pager frontmatter, and on terminal verdicts (`promoted`, `killed`) moves the folder to `review/` or `trash/`.

#### Status Values

| Status | Meaning | Command | Processed by |
|---|---|---|---|
| `brainstormed` | Raw idea, awaiting PM | `/agent-pm` | Product Engineer |
| `prd-ready` | PRD written, awaiting market analysis | `/agent-market` | Market Fit Analyzer |
| `market-ready` | Market analysis done, awaiting tech review | `/agent-tech` | Technical Lead |
| `tech-ready` | Solution doc written, awaiting financial review | `/agent-cfo` | CFO |
| `cfo-ready` | Financial assessment done, awaiting decision | `/agent-decision` | Decision Agent |
| `promoted` | Terminal — move to `review/` | N/A | Orchestrator |
| `killed` | Terminal — move to `trash/` | N/A | Orchestrator |

#### Notes Format (`notes.md`)

Append-only. Each agent appends a new entry:

```markdown
---
## [{Agent Role}] — {ISO timestamp}
**Verdict:** {pass | fail | delegate-back:{role} | refine}
**Next status:** {target status value}
**Iteration:** {n}
**Re-entry type:** {initial | human-feedback | decision-refinement | none}
**Pass type:** {full | refresh}

### Assessment
<!-- Agent's analysis relevant to their mandate. -->

### Concerns
<!-- Required if verdict is pass. Optional if fail. -->

### Failure Rationale
<!-- Required if verdict is fail. -->

### Delegation Rationale
<!-- Required if verdict is delegate-back. Which agent and why. -->

### Changes from prior iteration
<!-- Required on refresh passes and re-entries. What changed and why. -->
```

#### Delegation Mechanics

Any agent can delegate the idea back to a prior agent for refinement. When this happens:

1. The delegating agent appends their notes with `delegate-back:{role}` verdict.
2. The agent's notes entry includes `**Next status:**` set to the target agent's input status (e.g., `brainstormed` to route back to PM).
3. The orchestrator updates the one-pager frontmatter with the new status and increments `iteration`.
4. On the next processing pass (reconciliation loop or next nightly run), the target agent picks it up, reads the delegation rationale from notes, and re-processes.
5. **Circuit breaker:** If `iteration` exceeds the configurable max (default: `5`), the decision agent is invoked to force a promote-or-kill.

#### Context Management

Each agent invocation is stateless — Claude Code reads the idea folder from scratch every time. By the time an idea reaches the decision agent, the folder may contain six documents plus a notes file with entries from every prior agent and delegation cycle. Left unchecked, this can burn unnecessary tokens and dilute the agent's focus.

**Principle: each agent reads deeply what it owns, skims what it doesn't.**

Per-agent reading strategy:

| Agent | Read fully | Skim (headers + verdict summary) | Skip |
|---|---|---|---|
| Product Engineer | one-pager, feedback.md, notes.md | — | — |
| Market Fit Analyzer | one-pager, prd.md, notes.md | feedback.md | — |
| Technical Lead | one-pager, prd.md, market-analysis.md, notes.md | feedback.md | — |
| CFO | one-pager, notes.md, solution-doc.md (cost sections) | prd.md, market-analysis.md (pricing/revenue sections only) | solution-doc.md (architecture sections) |
| Decision Agent | one-pager, notes.md (all entries) | prd.md, market-analysis.md, solution-doc.md, financial-assessment.md (verdict summary sections only) | — |

"Skim" means the command file instructs the agent to read only the headers and the verdict/summary section of that document, not the full body. This keeps the context focused on what matters for the agent's mandate.

**Command file implementation.** Each command file should include explicit reading instructions:

```markdown
## Context Loading

1. Read `config/agent-profiles.md` — your role definition.
2. Read `$ARGUMENTS/one-pager.md` — full document.
3. Read `$ARGUMENTS/notes.md` — full document, pay attention to
   delegation rationale and concerns from prior agents.
4. Read `$ARGUMENTS/prd.md` — skim headers and "Verdict Summary" only.
   You don't need the full user stories.
5. Skip `$ARGUMENTS/solution-doc.md` — not yet created at your stage.
```

**Notes file growth.** The notes file is the most likely context bottleneck. On a contentious idea with 5 iterations, the notes file could have 10+ entries from different agents across multiple passes. The command file should instruct agents to focus on the most recent entry from each agent role, plus any delegation/refinement directives. Historical entries from the same agent on prior iterations can be skimmed for the verdict line only.

**Artifact size guidance.** Agents should keep their artifacts concise. Recommended ceilings:

| Artifact | Target length | Notes |
|---|---|---|
| One-pager | 300–500 words | Set by brainstorm agent |
| PRD | 800–1200 words | MVP scope, not a full spec |
| Market analysis | 600–1000 words | Data-driven, not narrative |
| Solution doc | 800–1200 words | Architecture sketch, not implementation detail |
| Financial assessment | 500–800 words | Numbers-heavy, prose-light |
| Notes entry (per agent) | 200–400 words | Assessment + concerns, not essays |

These aren't hard limits — an agent can exceed them if the idea warrants it — but they set expectations for what a normal artifact looks like. An agent producing a 3000-word PRD is overbuilding for this stage of ideation.

**Total context budget.** A fully-loaded idea folder at the decision agent stage, assuming artifacts at target lengths and 3 iterations of notes, should land around 6000–8000 tokens of content. Well within Sonnet's context window and cheap per-invocation. If an idea accumulates significantly more than this (e.g., 5 delegation cycles with lengthy notes), the decision agent should recognize bloat as a signal that the idea is too contentious and lean toward killing it.

### 3.4 Stage: Product Engineer (PM)

**Picks up status:** `brainstormed`
**Promotes to status:** `prd-ready`
**Mandate:** Transform a raw idea into a structured PRD. Assess whether the idea is well-formed enough to warrant further analysis.

**On pickup:**
1. Read the one-pager. Check for `feedback.md` — if present, this is a human-refined re-entry.
2. Evaluate: Is this idea coherent, specific enough, and non-trivially differentiated?
3. If **fail** → verdict `killed`.
4. If **pass** → write `prd.md` (or revise existing PRD if re-entry), append notes, verdict `prd-ready`.
5. If **delegate-back** → N/A (first agent in chain, cannot delegate backward; can only pass or fail).

**On re-entry with `feedback.md`:**
- Read all existing artifacts and the human feedback.
- Revise the PRD to address feedback. Do NOT start from scratch unless feedback explicitly requests it.
- Append notes explaining what changed and why.
- Mark the notes entry with `**Re-entry type:** human-feedback` so downstream agents know this is a refinement pass.

**On re-entry via decision agent surgical send-back:**
- Read the refinement directive in notes.
- Revise only the specified artifact gap.
- Mark the notes entry with `**Re-entry type:** decision-refinement`.

**PRD Format (`prd.md`):**

```markdown
# PRD: {Idea Title}

**Version:** {n}
**Author:** Product Engineer Agent
**Last Updated:** {ISO timestamp}

## Problem Definition
## Target Users & Personas
## User Stories / Jobs to Be Done
## Proposed Feature Set (MVP)
## Success Metrics
## Assumptions & Risks
## Out of Scope (v1)
## Open Questions for Market Fit
```

### 3.5 Stage: Market Fit Analyzer

**Picks up status:** `prd-ready`
**Promotes to status:** `market-ready`
**Mandate:** Assess whether a viable market exists. Pressure-test the target customer, pricing, and competitive landscape.

**On pickup:**
1. Read one-pager + PRD + notes.
2. Evaluate: Is there a reachable market? Is the pricing plausible? Is the competitive moat real?
3. If **fail** → verdict `killed`.
4. If **pass** → write `market-analysis.md`, verdict `market-ready`.
5. If **delegate-back:pm** → verdict with `next_status: brainstormed`.

**Market Analysis Format (`market-analysis.md`):**

```markdown
# Market Fit Analysis: {Idea Title}

**Version:** {n}
**Author:** Market Fit Analyzer Agent
**Last Updated:** {ISO timestamp}

## Market Size Estimate (TAM / SAM / SOM)
## Competitive Landscape
## Customer Acquisition Thesis
## Pricing Validation
## Go-to-Market Sketch
## Market Risks
## Verdict Summary
```

### 3.6 Stage: Technical Lead

**Picks up status:** `market-ready`
**Promotes to status:** `tech-ready`
**Mandate:** Determine technical viability. Estimate build cost. Produce a solution architecture doc if viable.

**On pickup:**
1. Read all prior artifacts + notes.
2. Evaluate: Can this be built? What's the stack? What's the estimated effort and infra cost?
3. If **fail** → verdict `killed`.
4. If **pass** → write `solution-doc.md`, verdict `tech-ready`.
5. If **delegate-back:{pm|market}** → verdict with `next_status` set to target agent's input status.

**Solution Doc Format (`solution-doc.md`):**

```markdown
# Solution Architecture: {Idea Title}

**Version:** {n}
**Author:** Technical Lead Agent
**Last Updated:** {ISO timestamp}

## Architecture Overview
## Tech Stack Recommendation
## Build vs Buy Decisions
## MVP Scope & Effort Estimate
## Infrastructure Cost Estimate (Monthly)
## Technical Risks & Unknowns
## Scalability Considerations
## Build Timeline (Rough)
```

### 3.7 Stage: CFO

**Picks up status:** `tech-ready`
**Promotes to status:** `cfo-ready`
**Mandate:** Financial viability. Unit economics, runway implications, break-even analysis.

**On pickup:**
1. Read all prior artifacts + notes (especially solution doc cost estimates and market analysis pricing).
2. Evaluate: Do the numbers work? What's the break-even? What's the capital requirement?
3. If **fail** → verdict `killed`.
4. If **pass** → write `financial-assessment.md`, verdict `cfo-ready`.
5. If **delegate-back:{pm|market|tech}** → verdict with `next_status` set to target agent's input status.

**Financial Assessment Format (`financial-assessment.md`):**

```markdown
# Financial Assessment: {Idea Title}

**Version:** {n}
**Author:** CFO Agent
**Last Updated:** {ISO timestamp}

## Unit Economics
## Revenue Projections (12-month, conservative)
## Cost Structure (Build + Run)
## Break-Even Analysis
## Capital Requirements
## Financial Risks
## ROI Assessment
## Verdict Summary
```

### 3.8 Stage: Decision Agent

**Picks up status:** `cfo-ready` (or any status when circuit breaker fires)
**Terminal statuses:** `promoted`, `killed`

**Mandate:** Read the full artifact bundle and all notes. Make a final promote-or-kill decision. When sending back, be surgical — route to the specific agent and artifact that would most change the outcome.

**Logic:**
1. Count pass/fail verdicts across all agents.
2. Read all concerns, even from passing agents.
3. If **promote** → verdict `promoted` (orchestrator moves folder to `review/`, appends to `changelog.md`).
4. If **kill** → verdict `killed` (orchestrator moves folder to `trash/`).
5. If **refine** → surgical send-back (see below). Only if circuit breaker has NOT fired; if it has, must promote or kill.

#### Surgical Send-Back

When the decision agent identifies a fixable weakness, it does NOT reset the idea to `brainstormed`. Instead:

1. **Identify the weakest artifact.** Which document has the gap — the PRD, market analysis, solution doc, or financial assessment?
2. **Route to the owning agent.** Set `next_status` to that agent's input status (e.g., `brainstormed` for PM, `prd-ready` for Market, `market-ready` for Tech, `tech-ready` for CFO).
3. **Write a refinement directive.** Append a structured entry to `notes.md`:

```markdown
---
## [Decision Agent — Refinement Directive] — {ISO timestamp}
**Verdict:** refine
**Target agent:** {role}
**Target artifact:** {artifact filename}
**Iteration:** {n}

### What needs to change
<!-- Specific, actionable description of the gap. -->

### What's working
<!-- Which artifacts and conclusions should be preserved. -->

### Downstream impact
<!-- If this artifact changes, which downstream artifacts may need a refresh pass. -->
```

4. **Downstream cascade.** When the target agent finishes their revision, all downstream agents re-evaluate with a lightweight "refresh" pass — they read the updated artifact and their own prior work, then either confirm their existing verdict or revise. They do NOT start from scratch. A refresh pass appends a new notes entry with `**Pass type:** refresh` to distinguish from a full evaluation.

This means a decision agent send-back to the PM doesn't wipe out the market analysis, solution doc, and financial assessment — it triggers targeted updates that ripple forward only as needed.

### 3.9 Stage: Human-Triggered Refinement

**Trigger:** Operator drops a `feedback.md` file into any idea folder in `review/` and moves the folder back to `pipeline/`, setting `status: brainstormed` in the one-pager frontmatter.

**Mandate:** Human feedback re-enters the pipeline as a first-class input. The pipeline treats it like a delegation from a senior stakeholder.

#### Workflow

1. Operator reads an idea in `review/`. It's close but has issues — pricing model is wrong, target customer is too broad, tech stack assumption is off.
2. Operator creates `feedback.md` in the idea folder:

```markdown
# Operator Feedback — {date}

## What needs to change
<!-- Specific issues. "The pricing model assumes enterprise buyers but the
     problem statement describes individual developers." -->

## What's good
<!-- What to preserve. "The technical approach is sound. Keep the solution doc." -->

## Constraints update
<!-- Any new constraints that apply to this idea specifically. Optional. -->
```

3. Operator runs `/send-back {idea-slug}` in Claude Code — or manually moves the folder from `review/` to `pipeline/` and sets `status: brainstormed` in the one-pager frontmatter.
4. On the next pipeline run, the PM agent picks it up. It sees `feedback.md` and all existing artifacts — this is NOT a fresh idea, it's a refinement request with human guidance.
5. The PM reads the feedback, revises the PRD accordingly, and the idea re-enters the pipeline. Downstream agents do refresh passes (same mechanic as decision agent surgical send-back).

#### Safeguards

- **Feedback.md is never overwritten by agents.** It's a read-only input from the operator's perspective. Agents can reference it in their notes but don't modify it.
- **Iteration counter increments.** The circuit breaker still applies. If the idea has already burned 4 iterations and the human sends it back, it gets one more agent pass before the circuit breaker forces a final decision.
- **Changelog annotation.** If a human-refined idea ultimately promotes, the changelog entry includes a `**Human-refined:** yes` flag so the operator can track which ideas needed manual intervention.

---

## 4. Changelog Format

**File:** `logs/changelog.md`

Append-only. One entry per promoted idea.

```markdown
## [{Idea Title}](./../review/{idea-slug}/) — {date}
**Iterations:** {n} | **Agents passed:** {list} | **Human-refined:** {yes/no}
**Elevator pitch:** {2-sentence summary from one-pager}
**Key concern:** {most significant concern from notes, if any}
**Refinement history:** {e.g., "Decision agent sent back to Tech Lead (solution doc cost estimate). 1 refresh cycle."}
**Estimated build:** {timeline from solution doc}
**Estimated monthly cost:** {from financial assessment}

---
```

This is the "skim file." It should be dense, scannable, and link to the full artifact bundle.

---

## 5. Agent Profile Definitions

**File:** `config/agent-profiles.md`

Each agent's profile includes:

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

### Personality Design

Agents should NOT be sycophantic. Each role has a built-in bias:

| Role | Bias | Pressure They Apply |
|---|---|---|
| Product Engineer | Builder optimism, tempered by scope awareness | "Is this idea specific enough to build? Is the scope realistic for MVP?" |
| Market Fit Analyzer | Skeptical realist | "Show me the customer. Show me they'll pay. Show me the moat." |
| Technical Lead | Pragmatic engineer | "Can we actually build this? What's the real cost? What breaks at scale?" |
| CFO | Conservative financial lens | "Do the numbers work with conservative assumptions? What's the burn?" |
| Decision Agent | Holistic tiebreaker | "Given all perspectives, is this worth a human's attention?" |

---

## 6. Configuration

**File:** `config/pipeline.config.json`

```json
{
  "schedule": {
    "cron": "0 2 * * *",
    "timezone": "America/Denver"
  },
  "brainstorm": {
    "ideas_per_run": 5,
    "raw_sketches": 10,
    "dedup_check": true,
    "self_critique": true
  },
  "agents": {
    "max_iterations_per_idea": 5,
    "circuit_breaker_action": "decision-agent",
    "max_reconciliation_loops": 2,
    "refresh_pass_on_upstream_change": true
  },
  "context": {
    "warn_total_tokens": 8000,
    "bloat_kill_signal": true
  },
  "model": {
    "brainstorm": "claude-sonnet-4-20250514",
    "agents": "claude-sonnet-4-20250514"
  },
  "logging": {
    "verbose": true,
    "metrics": true,
    "log_brainstorm_critique": true
  }
}
```

---

## 7. Execution Model

### 7.1 Everything Is a Command

There is no shell script orchestrator. The entire pipeline — dispatch, frontmatter updates, folder moves, logging — runs inside Claude Code via custom commands in `.claude/commands/`.

The only shell involved is the cron one-liner:

```
0 2 * * * cd /path/to/ideation-pipeline && claude -p "/run-pipeline" --model sonnet
```

Claude Code reads the `/run-pipeline` command, which tells it to scan `pipeline/`, read frontmatter, invoke agent commands as subagents, update state, move terminal folders, and write logs. No bash glue, no `jq`, no `grep`. Claude Code can read files, write files, and move directories natively.

### 7.2 The Orchestrator: `/run-pipeline`

The `/run-pipeline` command is itself a Claude Code custom command (`.claude/commands/run-pipeline.md`). Its prompt instructs Claude Code to:

1. Read `config/pipeline.config.json` for schedule, thresholds, and model settings.
2. Check `logs/runs/` — if a log exists for today's date, skip brainstorm. Otherwise invoke `/brainstorm` as a subagent (which reads the idea registry for dedup).
3. Scan every folder in `pipeline/`. For each idea, read the `status` and `iteration` fields from `one-pager.md` frontmatter. If an idea has no entry in `logs/idea-registry.md` (manual injection), append a registry entry before dispatching.
4. For each idea, spawn a subagent for the appropriate agent command based on status. Process in stage order (`brainstormed` first, then `prd-ready`, etc.) so an idea advanced in this run doesn't get double-processed.
5. After each subagent completes, read the verdict from `notes.md` (the agent's most recent entry) and update frontmatter accordingly.
6. Run reconciliation passes (up to N, default 2) for ideas whose status changed due to delegation or send-back.
7. Move terminal folders: `promoted` → `review/`, `killed` → `trash/`. Update registry entries with final outcome.
8. Append to `logs/changelog.md` for newly promoted ideas. Print `\a` to trigger terminal bell notification.
9. Write pipeline snapshot and run summary to `logs/runs/{date}.log.md`.

#### Subagent Architecture

The orchestrator stays lightweight. It holds only the dispatch state — which ideas are at which status, what verdicts came back, what needs reconciliation. Each agent invocation is a **subagent** with its own isolated context window.

```
/run-pipeline (orchestrator)
  │
  ├─ reads frontmatter, builds dispatch plan
  │
  ├─ spawns subagent: /brainstorm
  │   └─ isolated context: constraints + registry + brainstorm prompt
  │   └─ writes one-pagers to pipeline/, appends to registry
  │   └─ exits, context released
  │
  ├─ spawns subagent: /agent-pm pipeline/widget-builder
  │   └─ isolated context: idea folder + agent profile + constraints
  │   └─ writes prd.md, appends to notes.md
  │   └─ exits, context released
  │
  ├─ orchestrator reads notes.md verdict, updates frontmatter
  │
  ├─ spawns subagent: /agent-market pipeline/widget-builder
  │   └─ isolated context: idea folder + agent profile + constraints
  │   └─ ...
  │
  └─ (continues for each idea at each status)
```

This gives you three things: the orchestrator never bloats with artifact content from 20 different ideas, each agent gets a clean context focused on its one idea, and a failed subagent doesn't crash the orchestrator — it logs the error and moves to the next idea.

#### How the orchestrator reads verdicts

Agent subagents write their verdict into `notes.md` as part of the structured notes entry (the `**Verdict:**` and `**Next status:**` lines). After a subagent exits, the orchestrator reads the most recent notes entry to determine routing. This eliminates cross-context communication — the filesystem is the message bus.

The orchestrator updates frontmatter directly by rewriting the `**Status:**` and `**Iteration:**` lines in the one-pager. Claude Code can do this with a simple file read-write.

#### Orchestrator command sketch (`.claude/commands/run-pipeline.md`):

```markdown
# Pipeline Orchestrator

You are the orchestrator for the product ideation pipeline.
You coordinate the pipeline by spawning subagents for each task.
You do NOT process ideas yourself — you dispatch, read results,
and manage state.

Read `config/pipeline.config.json` for configuration.

## Phase 1: Brainstorm

Check if `logs/runs/{today's date}.log.md` exists.
- If yes: skip brainstorm, proceed to Phase 2.
- If no: spawn a subagent to run the brainstorm process. The
  subagent reads `config/constraints.md` and `logs/idea-registry.md`,
  generates ideas into `pipeline/`, and appends to the registry.
  Wait for the subagent to complete before proceeding.

## Phase 2: Dispatch

Scan every subfolder in `pipeline/`. For each idea:
1. Read `one-pager.md` frontmatter: extract `status` and `iteration`.
2. Check if the idea has an entry in `logs/idea-registry.md`. If not
   (manual injection), append a registry entry with the idea's slug,
   title, problem, keywords, and current status.
3. If `iteration` >= max_iterations from config, spawn a subagent for
   the decision agent with circuit-breaker flag. Skip normal dispatch.
4. Otherwise, spawn a subagent for the appropriate agent based on status:
   - `brainstormed` → spawn subagent as Product Engineer
   - `prd-ready` → spawn subagent as Market Fit Analyzer
   - `market-ready` → spawn subagent as Technical Lead
   - `tech-ready` → spawn subagent as CFO
   - `cfo-ready` → spawn subagent as Decision Agent
   - `promoted` → move folder to `review/` (no subagent needed)
   - `killed` → move folder to `trash/` (no subagent needed)
5. After each subagent completes, read the most recent entry in
   `notes.md` to get the verdict and next status.
6. Update the one-pager frontmatter with the new status.
   Increment iteration.

Process ideas in stage order: all `brainstormed` first, then
`prd-ready`, then `market-ready`, etc. This prevents double-processing.

## Phase 3: Reconciliation

Re-scan `pipeline/`. If any idea's status changed during Phase 2
(delegation or send-back), process it again. Repeat up to
{max_reconciliation_loops} times. Ideas still mid-cycle after that
wait for the next nightly run.

## Phase 4: Terminal moves

Scan `pipeline/` one final time. Move any `promoted` folders to
`review/`. Move any `killed` folders to `trash/`. Update each idea's
registry entry with the final outcome.

## Phase 5: Wrap-up

Write the pipeline snapshot and run log to `logs/runs/{date}.log.md`.
Append to `logs/changelog.md` for any newly promoted ideas.
Update `logs/metrics.json`.
```

### 7.3 Agent Commands

Agent prompts live in `.claude/commands/` as markdown files. Each is a self-contained role definition that the orchestrator spawns as a subagent, or that you can invoke directly as a slash command during interactive testing. The `$ARGUMENTS` variable is replaced with whatever follows the command name.

#### Agent Command Template

```markdown
# {Agent Role}

Read `config/agent-profiles.md` for your full role definition and evaluation criteria.
Read `config/constraints.md` for the current ideation constraints.

## Context Loading

{Per-agent reading instructions — see Context Management in §3.3.
Specify which files to read fully, which to skim, and which to skip.}

## Task

Process the idea at `$ARGUMENTS`.

{Role-specific instructions: what to evaluate, what artifact to produce,
what re-entry and refresh pass behavior looks like.}

## Artifact Guidelines

Keep your artifact concise — see target lengths in Context Management.
You are producing an ideation-stage document, not a production spec.

## Output

Append your assessment to notes.md using the notes format defined in the
agent profiles doc. Your verdict line in the notes entry IS the output —
the orchestrator reads it to determine next status.
```

#### Command Files

| Command | File | Picks up status | Produces |
|---|---|---|---|
| `/run-pipeline` | `.claude/commands/run-pipeline.md` | N/A (orchestrator) | Run log, changelog entries |
| `/brainstorm` | `.claude/commands/brainstorm.md` | N/A (generates new ideas) | One-pagers in `pipeline/` |
| `/agent-pm` | `.claude/commands/agent-pm.md` | `brainstormed` | `prd.md` |
| `/agent-market` | `.claude/commands/agent-market.md` | `prd-ready` | `market-analysis.md` |
| `/agent-tech` | `.claude/commands/agent-tech.md` | `market-ready` | `solution-doc.md` |
| `/agent-cfo` | `.claude/commands/agent-cfo.md` | `tech-ready` | `financial-assessment.md` |
| `/agent-decision` | `.claude/commands/agent-decision.md` | `cfo-ready` | Verdict only (promote/kill/refine) |
| `/send-back` | `.claude/commands/send-back.md` | N/A (helper) | Moves idea from `review/` to `pipeline/` |

#### Interactive Testing

You can test any command interactively by opening Claude Code in the project directory:

```
> /agent-pm pipeline/widget-builder
> /agent-market pipeline/widget-builder
> /agent-decision pipeline/widget-builder
> /brainstorm
> /run-pipeline
> /send-back widget-builder
```

Same prompts, same behavior, no test harness. This is the primary advantage of the all-commands architecture — everything works identically in batch (cron) and interactive (development) mode.

### 7.4 Multi-Night Progression

The pipeline is a **nightly batch processor with rolling state**. Each run advances every active idea by one step. Ideas brainstormed on Monday might not promote or die until Wednesday or Thursday. Multiple generations of ideas coexist in `pipeline/` at different stages, each progressing at its own pace.

A typical idea lifecycle across nightly runs:

| Night | What happens | Status after |
|---|---|---|
| Monday | Brainstormed, self-critique passes | `brainstormed` |
| Tuesday | PM writes PRD | `prd-ready` |
| Wednesday | Market agent analyzes, delegates back to PM | `brainstormed` (iteration 2) |
| Thursday | PM revises PRD per market feedback | `prd-ready` (iteration 3) |
| Thursday (reconciliation) | Market agent re-evaluates, passes | `market-ready` (iteration 3) |
| Friday | Tech lead writes solution doc | `tech-ready` |
| Saturday | CFO reviews financials | `cfo-ready` |
| Sunday | Decision agent promotes | `promoted` → moved to `review/` |

A clean idea with no delegations can promote in 5 nights. A contentious idea may take a full week. Ideas that hit the circuit breaker get force-resolved regardless.

### 7.5 Pipeline Snapshot

At the end of every run, the orchestrator writes a snapshot to the run log:

```markdown
## Pipeline Snapshot — {date}

| Status | Count | Ideas |
|---|---|---|
| brainstormed | 5 | widget-builder, api-monitor, ... |
| prd-ready | 3 | log-aggregator, ... |
| market-ready | 1 | email-scheduler |
| tech-ready | 2 | form-builder, auth-proxy |
| cfo-ready | 0 | — |

**review/:** 2 ideas awaiting human review
**trash/:** 7 ideas killed this week
**Total iterations burned tonight:** 14
**Reconciliation loops used:** 1 of 2
**Registry entries:** 47 total (12 promoted, 28 killed, 4 duplicate, 3 active)
**Thematic saturation:** none (or: WARNING — 4/10 sketches matched existing registry entries in "monitoring" space)
```

This is the health check. If `brainstormed` keeps growing but nothing reaches `prd-ready`, the PM agent is too aggressive. If everything clears PM but dies at market fit, the brainstorm constraints need tightening.

### 7.6 Delegation Re-Processing

When an agent delegates back or the decision agent sends back, the idea's status changes immediately but it won't be re-processed until the next dispatch pass. The reconciliation phase handles this — the orchestrator re-scans `pipeline/` and processes any ideas whose status changed during the current run. Up to N reconciliation loops per run (default 2). Ideas still in-flight after reconciliation loops simply wait for the next nightly run.

---

## 8. Error Handling

| Scenario | Behavior |
|---|---|
| Agent subagent fails mid-processing | Orchestrator logs error, skips idea, continues pipeline. Status unchanged — idea retries next run. Subagent failure does not crash the orchestrator. |
| Agent produces malformed notes entry | Orchestrator logs warning, treats as no-op. Status unchanged — idea retries next run. |
| Constraints file missing | Abort brainstorm phase. Process existing ideas in `pipeline/`. Log error. |
| Folder permissions error | Abort run. Log error with details. |
| Duplicate idea detected during brainstorm | Flag as duplicate in idea registry. Generate replacement sketch. Max 3 retries per slot, then accept whatever is generated. |
| Thematic saturation (3+ sketches match registry) | Log warning in run log and pipeline snapshot. Flag to operator that constraints may need broadening. Pipeline continues normally. |
| Idea registry file missing or corrupt | Brainstorm proceeds without dedup (log warning). Orchestrator recreates registry from existing idea folders on next run. |
| Manual injection duplicates existing idea | Orchestrator adds registry entry and lets PM agent process normally. PM agent may independently kill it if it recognizes the overlap from reading existing artifacts in other idea folders. |
| Circuit breaker fires | Decision agent forced invocation. Must output `promoted` or `killed` — no send-back. |
| Refresh pass agent disagrees after upstream change | Agent can escalate to decision agent with a `delegate-back:decision` verdict. Decision agent re-evaluates with the conflict context. |
| Malformed `feedback.md` | PM agent treats it as freeform input and does best-effort interpretation. Logs a warning suggesting the operator use the recommended format. |
| Re-entry idea hits circuit breaker immediately | If the idea re-enters with 4 prior iterations, it gets exactly 1 pass. If still unresolved, decision agent forces promote or kill. |
| Frontmatter status unrecognized | Log error, skip idea. Do not process ideas with unknown status values. |
| Idea folder exceeds context budget | Log warning with token estimate. If `bloat_kill_signal` is enabled in config, the decision agent treats excessive context as a negative signal — the idea is too contentious to survive. |
| Orchestrator hits Claude Code usage limit mid-run | Ideas already processed in this run keep their updated status. Unprocessed ideas stay at their current status for next run. Run log captures partial progress. |

---

## 9. Human Interface

The operator interacts with the pipeline through the filesystem:

| Action | How |
|---|---|
| Read promoted ideas | Check `review/` folder and `logs/changelog.md` |
| Understand rejections | Check `trash/` folder, read `notes.md` for kill rationale |
| Check pipeline health | Read the pipeline snapshot in `logs/runs/{date}.log.md` |
| Send idea back with feedback | Create `feedback.md` in the idea folder, then run `/send-back {idea-slug}` in Claude Code |
| Tune ideation focus | Edit `config/constraints.md` |
| Adjust agent behavior | Edit `config/agent-profiles.md` |
| Change pipeline knobs | Edit `config/pipeline.config.json` |
| Archive reviewed ideas | Move from `review/` to `archive/` |
| Resurrect trashed idea | Move from `trash/` back to `pipeline/`, set `status: brainstormed` |
| Inject own idea | Create a new idea folder in `pipeline/` with a one-pager (use the template), set `status: brainstormed` |
| Force re-run | Delete today's run log and run `claude -p "/run-pipeline"` |
| Run single agent manually | Open Claude Code, type `/agent-pm pipeline/{idea-slug}` |
| Monitor health | Read `logs/runs/{date}.log.md` and `logs/metrics.json` |
| Browse idea history | Read `logs/idea-registry.md` — every idea ever generated, with outcome and keywords |

---

## 10. Future Considerations (Out of Scope for v1)

These are not part of the initial build but are natural extensions:

- **Web search integration.** Let the market fit agent and brainstorm agent access web search for real competitive data and trend validation.
- **Git-backed state.** Commit each state transition so the full history is version-controlled and diffable.
- **Jira integration.** Promoted ideas auto-create Jira tickets, fitting into the existing Claude Code orchestration workflow.
- **Parallel agent execution.** Run agents concurrently using git worktrees (one worktree per idea in a stage).
- **Feedback loop.** ~~When the operator reviews ideas in `review/`, their accept/reject/notes feed back into the constraints file or agent profiles to improve future runs.~~ *Partially addressed in v0.2 via human-triggered refinement (`feedback.md` re-entry). Constraints/profile auto-evolution remains out of scope.*
- **Scoring model.** Numeric confidence scores from each agent, aggregated into a composite score for prioritization within the review folder.
- **Multi-model routing.** Use heavier models (Opus) for the decision agent and lighter models (Haiku) for initial brainstorm dedup checks.

---

## 11. Open Questions

1. ~~**Should the brainstorm agent have memory of past ideas?**~~ *Resolved: The idea registry (`logs/idea-registry.md`) serves as institutional memory. Keyword-based semantic matching replaces slug-only dedup. Thematic saturation warnings flag when the idea space is exhausted.*
2. ~~**How opinionated should agent profiles be?**~~ *Resolved: Generic. Agent profiles define mandate, evaluation criteria, and tone — not domain-specific heuristics. The constraints file handles domain focus. This keeps the agents reusable across different constraint configurations without rewriting profiles every time you shift ideation focus.*
3. ~~**Should the pipeline support manual injection?**~~ *Resolved: Yes, via `feedback.md` re-entry, manual injection into `pipeline/`, and the "move from trash to pipeline" pattern.*
4. ~~**Notification on completion?**~~ *Resolved: Terminal bell. The orchestrator prints `\a` after promoting ideas to `review/`, triggering the existing bell notification hook.*
5. ~~**Should trashed ideas ever auto-resurface?**~~ *Resolved: No. If constraints change significantly, the operator wipes the trash folder manually. The idea registry still remembers them, but the brainstorm agent can generate fresh takes on the same problem space since the constraints have shifted.*
6. ~~**Refresh pass depth.**~~ *Resolved: Selective refresh. The decision agent's refinement directive includes a `### Downstream impact` section that specifies which downstream agents need a refresh pass based on what changed. Agents not named in the downstream impact list keep their existing verdict. This saves tokens at the cost of requiring the decision agent to predict impact correctly — but if it gets it wrong, the next nightly run's agents will catch the inconsistency during their normal processing.*
7. ~~**Self-critique calibration.**~~ *Resolved: Configurable via `brainstorm.raw_sketches` in `pipeline.config.json`. The ratio of raw sketches to `ideas_per_run` determines cull rate. Default is 10:5 (50%). Operator can tune based on observed output quality.*
8. ~~**Human feedback format enforcement.**~~ *Resolved: Freeform. The PM agent can handle unstructured feedback. The suggested template in §3.9 is guidance, not a schema. Lowering friction for the operator matters more than making parsing easier for the agent.*
9. ~~**Orchestrator context window.**~~ *Resolved: The orchestrator uses subagents. Each idea is processed in an isolated subagent context, keeping the orchestrator's own context clean for routing and state management.*
10. ~~**Subagent vs. inline processing.**~~ *Resolved: Subagents. Each agent invocation spawns a subagent with its own context window. The orchestrator stays lightweight — it only holds frontmatter status, the dispatch loop, and verdict results. No risk of context pollution across ideas. Per-invocation overhead is acceptable given that runs happen at 2am with no time pressure.*
11. ~~**Registry context scaling.**~~ *Resolved for v1: Keep it flat. The registry runs at 2am on off-peak tokens — the context cost is acceptable. If it becomes a problem after months of runs, optimize in v2.*
12. ~~**Semantic dedup threshold.**~~ *Resolved for v1: Keep dedup lightweight — keyword overlap and title similarity only. Don't burn tokens on deep semantic comparison. If a near-duplicate slips through, the PM agent will likely kill it anyway. Fine-tune dedup aggressiveness in v2 after observing real output patterns.*
