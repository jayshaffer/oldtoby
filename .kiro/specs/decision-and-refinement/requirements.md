# Decision Agent and Refinement Requirements

## User Story 1: Final Decision
As the pipeline, I want a decision agent that makes a final promote-or-kill decision based on the full artifact bundle,
so that only ideas worth a human's attention reach the review folder.

### Acceptance Criteria
- WHEN an idea has `status: cfo-ready`, THE SYSTEM SHALL invoke `/agent-decision` with the idea folder path.
- WHEN processing, THE SYSTEM SHALL read `one-pager.md` and `notes.md` (all entries) fully, and skim `prd.md`, `market-analysis.md`, `solution-doc.md`, and `financial-assessment.md` (verdict summary sections only).
- WHEN the idea warrants human attention, THE SYSTEM SHALL set verdict to `promoted` with `Next status: promoted`.
- WHEN the idea does not warrant human attention, THE SYSTEM SHALL set verdict to `killed` with `Next status: killed`.
- WHEN the decision agent evaluates, THE SYSTEM SHALL count pass/fail verdicts across all agents and read all concerns, even from passing agents.

## User Story 2: Surgical Send-Back
As the pipeline, I want the decision agent to surgically route fixable ideas back to the specific agent that can address the weakness,
so that good ideas with one weak artifact get refined rather than killed.

### Acceptance Criteria
- WHEN the decision agent identifies a fixable weakness, THE SYSTEM SHALL set verdict to `refine` and identify the weakest artifact.
- WHEN sending back, THE SYSTEM SHALL route to the owning agent by setting `Next status` to that agent's input status (e.g., `brainstormed` for PM, `prd-ready` for Market, `market-ready` for Tech, `tech-ready` for CFO).
- WHEN sending back, THE SYSTEM SHALL append a structured refinement directive to `notes.md` with: target agent, target artifact, what needs to change, what's working, and downstream impact.
- WHEN the circuit breaker has fired (iteration >= max), THE SYSTEM SHALL NOT send back — it must promote or kill.

## User Story 3: Downstream Cascade After Send-Back
As the pipeline, I want downstream agents to do lightweight refresh passes after an upstream artifact changes,
so that existing analysis is preserved unless the change actually invalidates it.

### Acceptance Criteria
- WHEN the target agent finishes their revision after a send-back, THE SYSTEM SHALL trigger refresh passes for downstream agents specified in the refinement directive's `Downstream impact` section.
- WHEN doing a refresh pass, THE SYSTEM SHALL read the updated artifact and their own prior work, then confirm or revise their verdict.
- WHEN doing a refresh pass, THE SYSTEM SHALL NOT start their artifact from scratch — they update only what's affected by the upstream change.
- WHEN a refresh pass agent disagrees with the upstream change, THE SYSTEM SHALL escalate to the decision agent with a `delegate-back:decision` verdict.

## User Story 4: Circuit Breaker
As the pipeline, I want a circuit breaker that forces resolution on ideas that cycle too many times,
so that contentious ideas don't burn infinite pipeline iterations.

### Acceptance Criteria
- WHEN an idea's `iteration` exceeds `max_iterations_per_idea` from config (default 5), THE SYSTEM SHALL invoke the decision agent with a circuit-breaker flag.
- WHEN the circuit breaker fires, THE SYSTEM SHALL force the decision agent to output `promoted` or `killed` — no send-back allowed.
- WHEN an idea accumulates context significantly beyond the total context budget (6000-8000 tokens at decision stage), AND `bloat_kill_signal` is enabled in config, THE SYSTEM SHALL treat excessive context as a negative signal — the idea is too contentious to survive.

## User Story 5: Human-Triggered Refinement
As a product ideation operator, I want to send ideas back from review with feedback,
so that promising-but-flawed ideas get another pass through the pipeline with my guidance.

### Acceptance Criteria
- WHEN the operator creates `feedback.md` in an idea folder and runs `/send-back {slug}`, THE SYSTEM SHALL move the folder from `review/` to `pipeline/` and set `status: brainstormed`.
- WHEN `feedback.md` exists, THE SYSTEM SHALL treat it as a read-only input — agents can reference it in notes but SHALL NOT modify it.
- WHEN a human-refined idea enters the pipeline, THE SYSTEM SHALL increment the iteration counter. The circuit breaker still applies.
- WHEN a human-refined idea ultimately promotes, THE SYSTEM SHALL include `Human-refined: yes` in the changelog entry.
- WHEN the PM agent picks up a re-entered idea with `feedback.md`, THE SYSTEM SHALL revise the existing PRD to address feedback, not start from scratch (unless feedback explicitly requests it).

## User Story 6: Decision Agent Personality
As the pipeline, I want the decision agent to act as a holistic tiebreaker,
so that the final gate considers all perspectives without defaulting to any single agent's bias.

### Acceptance Criteria
- WHEN evaluating, THE SYSTEM SHALL apply holistic tiebreaker perspective — "Given all perspectives, is this worth a human's attention?"
- WHEN sending back, THE SYSTEM SHALL be surgical — route to the specific agent and artifact that would most change the outcome.
