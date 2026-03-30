# Decision Agent & Refinement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the decision agent (promote/kill/send-back with circuit breaker) and the human send-back helper command.

**Architecture:** Two Claude Code custom command files: `agent-decision.md` synthesizes all agent verdicts to make a final call (promote, kill, or surgical send-back), and `send-back.md` handles human-triggered refinement by moving ideas from `review/` back to `pipeline/`. The decision agent reads the full artifact bundle but only skims non-notes files. Downstream cascade (refresh pass routing) is handled by the orchestrator in Phase 5 — the decision agent's job is to specify which agents need refresh in its refinement directive.

**Tech Stack:** Claude Code custom commands (markdown prompt files), filesystem operations via Claude tool use.

---

## File Structure

| File | Responsibility |
|------|---------------|
| `.claude/commands/agent-decision.md` | Decision agent — synthesize verdicts, promote/kill/send-back, circuit breaker, bloat detection |
| `.claude/commands/send-back.md` | Human send-back helper — move idea from review/ to pipeline/, verify feedback.md |

---

### Task 1: Create the Decision agent command

**Files:**
- Create: `.claude/commands/agent-decision.md`

The Decision agent is the final gate. It picks up `cfo-ready` ideas, reads the full artifact bundle, counts verdicts, aggregates concerns, and makes a terminal decision (promote or kill) or routes back for targeted refinement (surgical send-back). It also enforces the circuit breaker and detects context bloat.

- [ ] **Step 1: Write `.claude/commands/agent-decision.md`**

Create the file with the following complete content:

```markdown
# Decision Agent

You are the Decision Agent for the OldToby ideation pipeline. Your job is to make the final call — promote this idea to human review, kill it, or surgically send it back for targeted refinement. Do not ask for input. Do not pause. Execute all steps in order.

---

## Step 1 — Load Context

Read the following files from the idea folder at `$ARGUMENTS`:

1. **`config/agent-profiles.md`** — Find the "Decision Agent" section. This is your role definition, mandate, and tone.
2. **`config/constraints.md`** — The operator's domain focus and market preferences.
3. **`config/pipeline.config.json`** — Read `agents.max_iterations_per_idea` (default 5) and `context.bloat_kill_signal` (default true).
4. **`$ARGUMENTS/one-pager.md`** — Read fully. Note the `iteration` value from frontmatter.
5. **`$ARGUMENTS/notes.md`** — Read fully. This is your primary input — it contains every agent's verdict, assessment, concerns, and any prior decision agent entries. If `notes.md` contains more than 5 entries, focus on the most recent entry from each agent role and all delegation/refinement directives — skim older entries for verdict lines only.
6. **`$ARGUMENTS/prd.md`** — Skim only: read the Problem Definition and Open Questions sections. Skip the rest.
7. **`$ARGUMENTS/market-analysis.md`** — Skim only: read the Verdict Summary section. Skip the rest.
8. **`$ARGUMENTS/solution-doc.md`** — Skim only: read the Technical Risks & Unknowns and MVP Scope & Effort Estimate sections. Skip the rest.
9. **`$ARGUMENTS/financial-assessment.md`** — Skim only: read the Verdict Summary and Financial Risks sections. Skip the rest.

**If `one-pager.md` or `notes.md` does not exist, abort immediately and report the error.**

---

## Step 2 — Check Circuit Breaker

Read the `iteration` value from `one-pager.md` frontmatter and `agents.max_iterations_per_idea` from `config/pipeline.config.json`.

**If `iteration >= max_iterations_per_idea`:** The circuit breaker has fired. You MUST output `promoted` or `killed`. Send-back is NOT allowed. Note this constraint — it overrides your normal judgment. If the idea has any merit at all, promote it. If it's fundamentally broken after this many iterations, kill it.

**If the circuit breaker has NOT fired:** All three verdicts (promote, kill, send-back) are available.

---

## Step 3 — Check Context Bloat

Read `context.bloat_kill_signal` from `config/pipeline.config.json`.

**If `bloat_kill_signal` is `true`:** Estimate the total token count of all files in the idea folder (one-pager + notes + all artifacts). Use a rough heuristic: ~1 token per word, count total words across all files. If the total exceeds ~8000 tokens, treat this as a negative signal. The idea has consumed excessive pipeline resources — it's too contentious or complex to survive the gauntlet cleanly. This is a signal, not an automatic kill — weigh it alongside the verdicts.

**If `bloat_kill_signal` is `false`:** Skip this check.

---

## Step 4 — Analyze Verdicts

Tally the verdicts from all agent entries in `notes.md`:

1. **Count pass vs fail** across all agents (Product Engineer, Market Fit Analyzer, Technical Lead, CFO). Ignore prior Decision Agent entries.
2. **Read ALL concerns** from every agent, including passing agents. Concerns from passing agents often contain the most important risk signals.
3. **Identify consensus strength:**
   - **Strong consensus (4/4 pass):** Lean promote unless concerns are dealbreakers.
   - **Majority pass (3/4 pass):** Evaluate whether the failing agent's concern is fixable or fundamental.
   - **Split (2/2):** Evaluate which agents' concerns are more critical. Lean toward send-back if fixable, kill if fundamental.
   - **Majority fail (3/4 or 4/4 fail):** Lean kill unless there's a clear single-point-of-failure that a send-back could fix.

4. **Check for prior send-backs:** If `notes.md` contains prior `[Decision Agent — Refinement Directive]` entries, read them. Has the idea already been sent back? Did the revision address the concern? Repeated send-backs for the same issue are a kill signal.

**Your tone:** Holistic tiebreaker. You weigh all perspectives fairly but aren't afraid to kill ideas that don't deserve more cycles. "Given all perspectives, is this worth a human's attention?"

---

## Step 5 — Make Decision

Choose one of three verdicts:

### Promote
**When:** The idea has earned human attention. Strong or majority pass, remaining concerns are manageable risks rather than dealbreakers. The artifact bundle tells a coherent story from problem through financials.

### Kill
**When:** The idea is fundamentally flawed. Majority fail, or concerns are dealbreakers that no amount of refinement can fix. Also: circuit breaker fired and idea lacks merit, or context bloat is extreme and concerns remain unresolved after multiple iterations.

### Send-Back (Surgical Refinement)
**When:** The idea has merit but one specific artifact is weak enough to change the outcome. Only available if circuit breaker has NOT fired.

To send back, you must identify:
- **Target agent:** Which agent owns the weakest artifact? Route to them.
- **Target artifact:** Which specific file needs revision?
- **What needs to change:** Be specific and actionable. Name the exact gap.
- **What's working:** Which artifacts and conclusions should be preserved.
- **Downstream impact:** If this artifact changes, which downstream agents need a refresh pass? Only name agents whose work depends on the artifact being revised.

Set `Next status` to the target agent's input status:
- PM → `brainstormed`
- Market → `prd-ready`
- Tech → `market-ready`
- CFO → `tech-ready`

---

## Step 6 — Append to Notes

Append a structured entry to `$ARGUMENTS/notes.md`. **This file is append-only — do not modify or delete any existing content.**

**For promote or kill verdicts:**

```
---
## [Decision Agent] — {ISO 8601 timestamp}
**Verdict:** {promoted | killed}
**Next status:** {promoted | killed}
**Iteration:** {current iteration number from one-pager.md frontmatter + 1}
**Circuit breaker:** {yes | no}

### Rationale
{Why this decision? Synthesize the agent verdicts, weigh the concerns, and explain the call. Reference specific agent assessments. 3-5 sentences.}

### Verdict Summary
{One-paragraph executive summary of the idea's strengths and weaknesses. This is what the human reviewer will read first if promoted.}
```

**For send-back (refinement directive):**

```
---
## [Decision Agent — Refinement Directive] — {ISO 8601 timestamp}
**Verdict:** refine
**Target agent:** {Product Engineer | Market Fit Analyzer | Technical Lead | CFO}
**Target artifact:** {prd.md | market-analysis.md | solution-doc.md | financial-assessment.md}
**Next status:** {brainstormed | prd-ready | market-ready | tech-ready}
**Iteration:** {current iteration number from one-pager.md frontmatter + 1}

### What needs to change
{Specific, actionable description of the gap. Name the exact section or conclusion that needs revision. 2-4 sentences.}

### What's working
{Which artifacts and conclusions should be preserved. Name specific files and sections. 2-3 sentences.}

### Downstream impact
{List which downstream agents need a refresh pass after this revision. Only name agents whose work directly depends on the artifact being revised. If none, say "None — revision is isolated."}
```

**Length:** 200-400 words for the notes entry.

**Do not modify `one-pager.md` frontmatter.** The orchestrator handles status and iteration updates.

If `notes.md` already exceeds 3000 words, keep your entry concise — favor the lower end of the 200-400 word range.

---

## Step 7 — Output Summary

Print a plain-text summary to stdout:

```
Decision Agent complete.
Idea: {slug from one-pager.md frontmatter}
Verdict: {promoted | killed | refine}
Next status: {promoted | killed | brainstormed | prd-ready | market-ready | tech-ready}
Circuit breaker: {yes | no}
```

If verdict is `killed`, add:
```
Reason: {single sentence explaining why}
```

If verdict is `refine`, add:
```
Target: {agent role} — {artifact filename}
Gap: {single sentence describing what needs to change}
```
```

- [ ] **Step 2: Review the command file**

Read back `.claude/commands/agent-decision.md` and verify:
- All 7 steps are present and complete
- Context loading matches spec (one-pager + notes.md fully, all other artifacts skimmed at verdict/summary sections only)
- Circuit breaker logic reads `max_iterations_per_idea` from config and disables send-back when fired
- Context bloat detection reads `bloat_kill_signal` from config and estimates tokens (~8000 threshold)
- Verdict counting tallies pass/fail across 4 agents, reads all concerns
- Promote/kill/send-back logic is complete with correct conditions
- Surgical send-back includes all 5 fields: target agent, target artifact, what needs to change, what's working, downstream impact
- Refinement directive format matches design doc
- Notes format distinguishes promote/kill entries from refinement directive entries
- Frontmatter guard and notes growth management are present

- [ ] **Step 3: Commit**

```bash
git add .claude/commands/agent-decision.md
git commit -m "feat: add Decision agent command with promote/kill/send-back"
```

---

### Task 2: Create the Send-back helper command

**Files:**
- Create: `.claude/commands/send-back.md`

The send-back command handles human-triggered refinement. It moves an idea from `review/` back to `pipeline/`, resets its status, and verifies that `feedback.md` exists so the PM agent knows what to revise.

- [ ] **Step 1: Write `.claude/commands/send-back.md`**

Create the file with the following complete content:

```markdown
# Send-Back Helper

You are executing a human-triggered send-back for the OldToby ideation pipeline. The operator wants to send a promoted idea back into the pipeline for another pass with their feedback. Do not ask for input. Do not pause. Execute all steps in order.

The idea slug is provided as `$ARGUMENTS`.

---

## Step 1 — Locate the Idea

Look for the idea folder at `review/$ARGUMENTS/`.

**If the folder does not exist at `review/$ARGUMENTS/`:** Check `pipeline/$ARGUMENTS/` — the idea may already be in the pipeline. If found there, report that the idea is already in the pipeline and abort. If not found in either location, report that the idea slug was not found and abort.

---

## Step 2 — Verify Feedback File

Check if `review/$ARGUMENTS/feedback.md` exists.

**If `feedback.md` exists:** Read it to confirm it has content. This is the operator's feedback that the PM agent will use to revise the PRD.

**If `feedback.md` does not exist:** Print a warning:
```
WARNING: No feedback.md found in review/{slug}/. The PM agent will re-evaluate the idea without specific human guidance. Consider creating feedback.md with your revision notes before running the pipeline.
```

Proceed anyway — the send-back is still valid without feedback.

---

## Step 3 — Move Idea to Pipeline

Move the entire idea folder from `review/$ARGUMENTS/` to `pipeline/$ARGUMENTS/`.

**Use the following approach:**
1. Create `pipeline/$ARGUMENTS/` if it doesn't exist
2. Copy all files from `review/$ARGUMENTS/` to `pipeline/$ARGUMENTS/`
3. Verify all files were copied successfully
4. Delete `review/$ARGUMENTS/` and its contents

---

## Step 4 — Update Frontmatter

Edit `pipeline/$ARGUMENTS/one-pager.md` and update the frontmatter:
- Set `status: brainstormed` — the idea re-enters at the beginning of the review chain
- Increment `iteration` by 1

Do NOT change any other frontmatter fields (slug, generated, keywords).

---

## Step 5 — Update Idea Registry

Append an update row to `logs/idea-registry.md` for this re-entered idea:

```
| {slug} | {title from one-pager} | {problem one-line from one-pager} | {keywords} | re-entered | {today's date YYYY-MM-DD} | Human send-back |
```

**Append only — do not modify existing rows.**

---

## Step 6 — Output Summary

Print a plain-text summary:

```
Send-back complete.
Idea: {slug}
Moved: review/{slug}/ → pipeline/{slug}/
Status: brainstormed
Iteration: {new iteration number}
Feedback: {present | missing (warning issued)}
Registry: updated
```
```

- [ ] **Step 2: Review the command file**

Read back `.claude/commands/send-back.md` and verify:
- All 6 steps are present and complete
- Locates idea in `review/` (not `pipeline/`)
- Checks for `feedback.md` and warns if missing (but proceeds)
- Moves folder from `review/` to `pipeline/`
- Updates frontmatter: status to `brainstormed`, increments iteration
- Updates idea-registry.md with re-entered status
- Output summary includes all relevant info

- [ ] **Step 3: Commit**

```bash
git add .claude/commands/send-back.md
git commit -m "feat: add send-back helper command for human refinement"
```

---

### Task 3: Create test fixture and validate decision agent

**Files:**
- Create: `pipeline/test-decision/` (test fixture with full artifact bundle)

This task creates a fully-formed idea with all 4 agent artifacts and runs the decision agent to validate promote behavior. Then tests send-back behavior.

- [ ] **Step 1: Create test fixture — a cfo-ready idea with full artifacts**

Create `pipeline/test-decision/one-pager.md`:

```markdown
---
slug: test-decision
generated: 2026-03-29T00:00:00Z
status: cfo-ready
iteration: 4
keywords: [ci-cd, build-cache, developer-tools, performance]
---

# Build Cache Analyzer

## Elevator Pitch
Engineering teams waste hours every week on slow CI/CD builds because they can't see which steps are cache-busting and why. Build Cache Analyzer monitors CI pipelines, identifies cache invalidation patterns, and recommends fixes to cut build times by 30-50%.

## Problem Statement
CI/CD build times creep up as codebases grow, but teams lack visibility into why. Build caches get invalidated by seemingly minor changes. Teams can't prioritize fixes. This wastes developer time and increases infrastructure costs.

## Proposed Solution
A lightweight agent that integrates with CI/CD platforms via their APIs. It monitors build logs, tracks cache hit/miss rates per step, identifies root causes of cache invalidations, and surfaces actionable recommendations.

## Target Customer
Platform engineers and DevOps leads at mid-size engineering teams (10-50 developers) running 50+ CI builds per day.

## Revenue Model
SaaS subscription. $49/month for teams up to 20 developers, $149/month for 20-50 developers.

## Differentiation
Purpose-built for cache optimization — tells you exactly why your cache is busting and how to fix it, unlike general CI/CD observability tools.

## Open Questions
1. How much access do CI/CD platform APIs provide to cache metadata?
2. What's the retention model once caches are optimized?
```

Create `pipeline/test-decision/prd.md`:

```markdown
# PRD: Build Cache Analyzer

## Problem Definition
CI/CD build times creep up silently as codebases grow. The root cause is often cache invalidation triggered by seemingly minor changes. Teams lack visibility into which changes cause the most cache busting.

## Target Users & Personas
Platform engineers at mid-size teams (10-50 devs) responsible for CI infrastructure and developer productivity.

## User Stories / JTBD
1. As a platform engineer, I want to see which CI steps have the lowest cache hit rates so I can prioritize fixes.
2. As a DevOps lead, I want alerts when a PR will invalidate critical caches so I can review the change.
3. As a developer, I want recommendations for how to restructure my Dockerfile to improve cache efficiency.

## Proposed Feature Set (MVP)
- GitHub Actions integration via GitHub App
- Cache hit/miss tracking per workflow step
- Root cause analysis for cache invalidations
- Actionable recommendations dashboard
- Weekly cache efficiency reports

## Success Metrics
- 30% reduction in average build time within 30 days of adoption
- 80% of recommendations acted upon within 1 week

## Assumptions & Risks
- GitHub Actions API exposes sufficient cache metadata
- Teams will install a GitHub App for monitoring

## Out of Scope (v1)
- GitLab CI, CircleCI support (v2)
- Custom CI system support
- Build optimization beyond caching

## Open Questions for Market Fit
1. Is cache optimization a standalone product or a feature?
2. What's the competitive risk from GitHub adding native cache analytics?
```

Create `pipeline/test-decision/market-analysis.md`:

```markdown
# Market Analysis: Build Cache Analyzer

## TAM/SAM/SOM
TAM: $2.1B (CI/CD tooling market). SAM: $210M (teams with 10-50 devs using GitHub Actions). SOM: $2.1M (1% of SAM in year 1).

## Competitive Landscape
Direct: None purpose-built for cache analytics. Adjacent: Datadog CI Visibility, BuildPulse (test-focused), Trunk.io (merge queues). Incumbents could add this as a feature.

## Customer Acquisition Thesis
GitHub Marketplace listing, developer blog content on CI optimization, conference talks. Path to first 100 customers through organic GitHub Marketplace discovery and targeted outreach to teams with >30min build times.

## Pricing Validation
$49-149/month aligns with developer tooling pricing. Comparable to Codecov ($29-99), BuildPulse ($50-200).

## Go-to-Market Sketch
Launch on GitHub Marketplace. Free tier for open source. Content marketing on CI optimization. Target teams posting about slow builds on Twitter/HN.

## Market Risks
GitHub could build native cache analytics (18-30 month window). Market may view this as a feature, not a product.

## Verdict Summary
Viable niche market. Clear pain point, reachable customers, defensible for 18-24 months. Retention risk after initial optimization is the key unknown.
```

Create `pipeline/test-decision/solution-doc.md`:

```markdown
# Solution Doc: Build Cache Analyzer

## Architecture Overview
GitHub App receives webhook events. Worker processes build logs asynchronously. PostgreSQL stores cache metrics. React dashboard serves recommendations.

## Tech Stack Recommendation
TypeScript/Node.js backend, React frontend, PostgreSQL, deployed on Railway or Render.

## Build vs Buy
Buy: Auth (Clerk), hosting (Railway), monitoring (Sentry). Build: Log parser, cache analysis engine, recommendation engine.

## MVP Scope & Effort Estimate
8-9 developer-weeks. Core: GitHub App + webhook handler (2w), log parser + cache analyzer (3w), dashboard + recommendations (2w), infrastructure + auth (1-2w).

## Infra Cost Estimate (Monthly)
~$50/month at launch (Railway starter, PostgreSQL, Sentry). Scales to ~$200/month at 100 customers.

## Technical Risks & Unknowns
GitHub Actions step logs may not expose structured cache metadata. Fallback: custom GitHub Action for cache instrumentation (adds onboarding friction).

## Scalability Considerations
Log processing is the bottleneck. Queue-based processing handles 10x. At 100x, need dedicated log infrastructure.

## Build Timeline (Rough)
MVP: 8-9 weeks. v1.1 (GitLab support): +4 weeks. v1.2 (advanced recommendations): +3 weeks.
```

Create `pipeline/test-decision/financial-assessment.md`:

```markdown
# Financial Assessment: Build Cache Analyzer

## Unit Economics
Revenue per customer: $75/month blended ARPU. Cost to serve: ~$6/month (infra + monitoring). Contribution margin: ~92%.

## Revenue Projections (12-Month Conservative)
Month 1-3: 5-15 customers ($375-$1,125 MRR). Month 4-6: 15-50 customers ($1,125-$3,750 MRR). Month 7-9: 50-100 customers ($3,750-$7,500 MRR). Month 10-12: 100-140 customers ($7,500-$10,500 MRR). Assumes 5% monthly churn, 8-20 new teams/month ramp.

## Cost Structure (Build + Run)
Build: ~$18,000 (9 dev-weeks at $2,000/week). Run: ~$50-200/month infra + $0/month labor (solo founder). Total first-year cost: ~$20,400.

## Break-Even Analysis
Break-even on monthly costs: month 4-5 (~20 customers). Break-even on investment: month 8-9 post-launch.

## Capital Requirements
~$26,500 to break-even. Self-fundable. No external capital required.

## Financial Risks
Churn risk if product is "one-time fix." LTV depends on ongoing value from regression alerts and trend monitoring. At 7%+ monthly churn, unit economics strain.

## ROI Assessment
12-month ROI: ~96% on invested capital. Payback period: 8-9 months.

## Verdict Summary
Numbers work with conservative assumptions. Self-fundable, positive unit economics from month 1. Primary risk is churn — monitor closely.
```

Create `pipeline/test-decision/notes.md` with 4 agent entries:

```markdown
---
## [Product Engineer] — 2026-03-29T00:00:00Z
**Verdict:** pass
**Next status:** prd-ready
**Iteration:** 1
**Re-entry type:** initial
**Pass type:** full

### Assessment
Build Cache Analyzer targets a specific, underserved pain point for platform engineers. The problem is concrete — CI build times creep up due to cache invalidation — and the solution is specific enough to scope an MVP.

### Concerns
Retention risk after initial cache optimization. The tool needs ongoing value beyond the first fix cycle — regression alerts and trend monitoring may address this but are unvalidated.

---
## [Market Fit Analyzer] — 2026-03-29T01:00:00Z
**Verdict:** pass
**Next status:** market-ready
**Iteration:** 2
**Re-entry type:** initial
**Pass type:** full

### Assessment
Viable niche market. GitHub Marketplace provides a concrete acquisition channel. Pricing aligns with developer tooling norms. No direct competitor in cache-specific analytics.

### Concerns
GitHub could build native cache analytics within 18-30 months. Retention cliff once initial optimization is complete. Market may view this as a feature rather than a standalone product.

---
## [Technical Lead] — 2026-03-29T02:00:00Z
**Verdict:** pass
**Next status:** tech-ready
**Iteration:** 3
**Re-entry type:** initial
**Pass type:** full

### Assessment
Technically straightforward. Standard web app stack with GitHub App integration. 8-9 developer-weeks for MVP. Infra costs are minimal ($50/month at launch).

### Concerns
GitHub Actions log parsing is the key technical risk. If structured cache metadata isn't available in step logs, a custom GitHub Action is needed — this adds onboarding friction.

---
## [CFO] — 2026-03-29T03:00:00Z
**Verdict:** pass
**Next status:** cfo-ready
**Iteration:** 4
**Re-entry type:** initial
**Pass type:** full

### Assessment
Numbers work with conservative assumptions. 92% contribution margin, self-fundable at ~$26,500, break-even by month 8-9. 12-month ROI of ~96%.

### Concerns
Churn is the critical variable. At 5% monthly churn the model is solid. At 7%+ it strains. Early churn measurement should be the primary leading indicator.
```

- [ ] **Step 2: Run Decision agent on test idea (expect promote)**

Run: `claude -p "/agent-decision pipeline/test-decision" --model sonnet --allowedTools "Write,Read,Edit,Glob,Grep,Bash"`

Verify:
- `pipeline/test-decision/notes.md` has a `[Decision Agent]` entry appended
- Verdict is `promoted` (4/4 pass, strong consensus)
- Notes entry has Rationale and Verdict Summary sections
- Circuit breaker field is `no` (iteration 4, max is 5)

- [ ] **Step 3: Test send-back helper**

First, move the test idea to `review/` to simulate a promoted idea:
```bash
mkdir -p review/
mv pipeline/test-decision review/test-decision
```

Create `review/test-decision/feedback.md`:
```markdown
# Operator Feedback — 2026-03-29

## What needs to change
The retention story is too weak. I need the PRD to include a concrete retention feature set — not just "regression alerts" as a hand-wave. What specific ongoing value does this provide after initial optimization?

## What's good
Market positioning is strong. Technical approach is sound. Financials work.

## Constraints update
None.
```

Run: `claude -p "/send-back test-decision" --model sonnet --allowedTools "Write,Read,Edit,Glob,Grep,Bash"`

Verify:
- Idea folder moved from `review/test-decision/` to `pipeline/test-decision/`
- `pipeline/test-decision/one-pager.md` frontmatter has `status: brainstormed` and iteration incremented
- `feedback.md` is present in `pipeline/test-decision/`
- `logs/idea-registry.md` has a new row with status `re-entered`
- Output summary shows feedback present

- [ ] **Step 4: Clean up test fixtures**

```bash
rm -rf pipeline/test-decision review/test-decision
```

- [ ] **Step 5: Commit test cleanup**

```bash
git add -A
git commit -m "chore: complete Phase 4 decision agent testing"
```

---

### Task 4: Update master task lists

**Files:**
- Modify: `.kiro/specs/tasks.md`
- Modify: `.kiro/specs/decision-and-refinement/tasks.md`

- [ ] **Step 1: Mark Phase 4 tasks as complete**

Update `.kiro/specs/tasks.md` — check off all Phase 4 items:
```markdown
- [x] Create `.claude/commands/agent-decision.md` (verdict counting, promote/kill/send-back, circuit breaker, bloat detection)
- [x] Create `.claude/commands/send-back.md` (move from review to pipeline, registry update)
- [x] Implement downstream cascade support (refresh pass routing, escalation)
- [x] Test decision agent (promote, send-back, circuit breaker, cascade, re-entry)
```

Update `.kiro/specs/decision-and-refinement/tasks.md` — check off all sub-tasks in sections 1-4.

Note: Downstream cascade (Task 3 in the spec) is specified in the decision agent's refinement directive via the `Downstream impact` section. The actual dispatch of refresh passes is the orchestrator's responsibility (Phase 5). The decision agent's job is complete — it tells the orchestrator which agents need refresh.

- [ ] **Step 2: Commit**

```bash
git add .kiro/specs/tasks.md .kiro/specs/decision-and-refinement/tasks.md
git commit -m "chore: mark Phase 4 decision agent tasks complete"
```
