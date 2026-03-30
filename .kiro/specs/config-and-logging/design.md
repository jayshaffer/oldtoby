# Configuration and Logging Design

## Architecture

### Configuration Files
- `config/pipeline.config.json` — All pipeline knobs (schedule, thresholds, models, logging)
- `config/agent-profiles.md` — Role definitions, mandates, evaluation criteria, personality
- `config/constraints.md` — Human-authored ideation focus (domain, tech, market, anti-patterns)

### Log Files
- `logs/runs/{YYYY-MM-DD}.log.md` — Per-run execution log with pipeline snapshot
- `logs/changelog.md` — Append-only promoted idea summaries
- `logs/idea-registry.md` — Append-only master index of every idea
- `logs/metrics.json` — Cumulative pipeline stats

## Data Models

### pipeline.config.json

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

### Idea Registry Table Format

```markdown
| Slug | Title | Problem (one-line) | Keywords | Status | Date | Outcome |
|---|---|---|---|---|---|---|
```

### Changelog Entry Format

```markdown
## [{Idea Title}](./../review/{idea-slug}/) — {date}
**Iterations:** {n} | **Agents passed:** {list} | **Human-refined:** {yes/no}
**Elevator pitch:** {2-sentence summary}
**Key concern:** {most significant concern from notes}
**Refinement history:** {e.g., "Decision agent sent back to Tech Lead. 1 refresh cycle."}
**Estimated build:** {timeline from solution doc}
**Estimated monthly cost:** {from financial assessment}
```

### Pipeline Snapshot Format

```markdown
## Pipeline Snapshot — {date}

| Status | Count | Ideas |
|---|---|---|
| brainstormed | N | slug1, slug2, ... |
| prd-ready | N | ... |
| market-ready | N | ... |
| tech-ready | N | ... |
| cfo-ready | N | ... |

**review/:** N ideas awaiting human review
**trash/:** N ideas killed this week
**Total iterations burned tonight:** N
**Reconciliation loops used:** N of max
**Registry entries:** N total (N promoted, N killed, N duplicate, N active)
**Thematic saturation:** none | WARNING — N/M sketches matched
```

### Metrics JSON Structure

```json
{
  "total_ideas_generated": 0,
  "total_promoted": 0,
  "total_killed": 0,
  "total_duplicates": 0,
  "pass_rates": {
    "pm": 0.0,
    "market": 0.0,
    "tech": 0.0,
    "cfo": 0.0,
    "decision": 0.0
  },
  "avg_iterations_to_promotion": 0.0,
  "avg_iterations_to_kill": 0.0,
  "last_updated": ""
}
```

## Key Design Goals

1. **Operator tunes via config, not code**: All pipeline behavior is controllable through `pipeline.config.json` and `constraints.md` without touching command files.
2. **Scannable outputs**: Changelog and pipeline snapshot are designed for quick human scanning, not detailed reading.
3. **Append-only logs**: Registry and changelog are never pruned. The run log is one file per day.
4. **Institutional memory**: The registry captures every idea across all outcomes, enabling dedup and thematic saturation detection across the full pipeline history.
