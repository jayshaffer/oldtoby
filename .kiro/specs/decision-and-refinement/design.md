# Decision Agent and Refinement Design

## Architecture

### Components
- `.claude/commands/agent-decision.md` — Decision agent command
- `.claude/commands/send-back.md` — Human refinement helper command

## Decision Agent Logic

```
1. Read full artifact bundle (one-pager, notes fully; all other artifacts skimmed)
2. Count pass/fail verdicts across all agents
3. Read ALL concerns, even from passing agents
4. Decision:
   a. Promote → verdict "promoted"
   b. Kill → verdict "killed"
   c. Refine → surgical send-back (only if circuit breaker NOT fired)
```

### Surgical Send-Back Flow

```
Decision agent identifies weakest artifact
  |
  +-- Sets Next status to owning agent's input status
  |     brainstormed (PM), prd-ready (Market), market-ready (Tech), tech-ready (CFO)
  |
  +-- Writes refinement directive to notes.md:
  |     - Target agent and artifact
  |     - What needs to change (specific, actionable)
  |     - What's working (preserve these)
  |     - Downstream impact (which agents need refresh)
  |
  +-- Orchestrator updates frontmatter, increments iteration
  |
  +-- Target agent picks up, reads directive, revises artifact
  |
  +-- Downstream agents do refresh passes (only those named in directive)
  |
  +-- Idea returns to decision agent
```

### Refinement Directive Format

```markdown
---
## [Decision Agent — Refinement Directive] — {ISO timestamp}
**Verdict:** refine
**Target agent:** {role}
**Target artifact:** {artifact filename}
**Iteration:** {n}

### What needs to change
{Specific, actionable description of the gap}

### What's working
{Which artifacts and conclusions should be preserved}

### Downstream impact
{If this artifact changes, which downstream artifacts may need refresh}
```

## Send-Back Helper

The `/send-back` command (`.claude/commands/send-back.md`) handles human-triggered refinement:

1. Accept idea slug as `$ARGUMENTS`
2. Move folder from `review/` to `pipeline/`
3. Set `status: brainstormed` in one-pager frontmatter
4. Verify `feedback.md` exists (warn if missing)
5. Update `logs/idea-registry.md` entry

### Feedback File Format

```markdown
# Operator Feedback — {date}

## What needs to change
{Specific issues}

## What's good
{What to preserve}

## Constraints update
{New constraints for this idea specifically. Optional.}
```

## Circuit Breaker

- Trigger: `iteration >= max_iterations_per_idea` (default 5)
- Behavior: Decision agent is invoked with force flag. Must output `promoted` or `killed`. No send-back.
- Context bloat signal: If `bloat_kill_signal` is enabled and the idea folder exceeds the context budget (~8000 tokens), excessive context is treated as a negative signal.

## Downstream Cascade

When a send-back revision completes:
- Only agents named in the `### Downstream impact` section do refresh passes
- Agents NOT named keep their existing verdict
- If a refresh agent disagrees, they escalate with `delegate-back:decision`
- This saves tokens at the cost of requiring the decision agent to correctly predict impact

## Key Design Goals

1. **Surgical refinement over wholesale rejection**: Route to the specific agent that can fix the issue, not back to the start.
2. **Bounded iteration**: Circuit breaker prevents infinite cycling. Context bloat is itself a kill signal.
3. **Human feedback as first-class input**: `feedback.md` re-enters the pipeline through the same mechanism as any other re-entry — agents read it, revise artifacts, cascade downstream.
4. **Preserve existing work**: Send-backs and refresh passes update, not replace. A decision agent send-back to PM doesn't wipe the market analysis, solution doc, or financial assessment.
