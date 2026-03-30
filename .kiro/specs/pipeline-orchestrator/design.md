# Pipeline Orchestrator Design

## Architecture

The orchestrator is a Claude Code custom command (`.claude/commands/run-pipeline.md`) — not a shell script. The only shell involved is a single cron one-liner that invokes Claude Code.

### Components
- **Cron entry**: `0 2 * * * cd /path/to/ideation-pipeline && claude -p "/run-pipeline" --model sonnet`
- **Orchestrator command**: `.claude/commands/run-pipeline.md` — the prompt that drives all dispatch logic
- **Subagent commands**: `.claude/commands/agent-*.md`, `.claude/commands/brainstorm.md` — each spawned as isolated subagents
- **Filesystem state**: `pipeline/`, `review/`, `trash/` folders; frontmatter `status` in `one-pager.md` is the routing key

### Execution Flow

```
/run-pipeline (orchestrator)
  |
  +-- Phase 1: Check run log -> conditionally spawn /brainstorm subagent
  |
  +-- Phase 2: Scan pipeline/, build dispatch plan by reading frontmatter
  |     |
  |     +-- For each idea (stage order): spawn appropriate agent subagent
  |     +-- After subagent exits: read notes.md verdict, update frontmatter
  |
  +-- Phase 3: Reconciliation (up to N loops for delegation/send-back)
  |
  +-- Phase 4: Terminal moves (promoted -> review/, killed -> trash/)
  |
  +-- Phase 5: Write run log, changelog, metrics
```

### Key Design Goals

1. **Orchestrator stays lightweight**: It holds only dispatch state (which ideas at which status, what verdicts came back). It never reads full artifacts — that's the agents' job.
2. **Filesystem as message bus**: Agents write verdicts to `notes.md`. The orchestrator reads them after subagent exit. No cross-context communication needed.
3. **Stage-ordered processing**: Process all ideas at earlier stages first to prevent an idea advanced in this run from being double-processed.
4. **Fault isolation**: Each subagent is independent. A failed subagent logs an error; the orchestrator continues with the next idea.

## Data Flow

### State Machine

```
brainstormed -> prd-ready -> market-ready -> tech-ready -> cfo-ready -> promoted (terminal)
     ^              |              |              |              |
     |              v              v              v              v
     +---- delegate-back:pm ------+              |              |
     +---- delegate-back:pm ----------------------+              |
     +---- delegate-back:pm --------------------------------------+
                                                                 |
                                                          killed (terminal)
```

Any agent can delegate back to a prior agent. The decision agent can surgically send back to any specific agent. The circuit breaker forces promote-or-kill when `iteration` exceeds the configured max.

### Frontmatter Contract

The orchestrator reads and writes these fields in `one-pager.md`:

```
**Status:** {brainstormed|prd-ready|market-ready|tech-ready|cfo-ready|promoted|killed}
**Iteration:** {integer}
```

### Verdict Contract

The orchestrator reads these lines from the most recent `notes.md` entry:

```
**Verdict:** {pass|fail|delegate-back:{role}|refine}
**Next status:** {target status value}
```

## Interfaces

### Input
- `config/pipeline.config.json` — schedule, thresholds, model settings
- `pipeline/*/one-pager.md` — frontmatter status for dispatch routing
- `pipeline/*/notes.md` — agent verdicts after subagent completion
- `logs/runs/{date}.log.md` — existence check for idempotency
- `logs/idea-registry.md` — for manual injection detection

### Output
- Updated `one-pager.md` frontmatter (status, iteration)
- Moved folders (`pipeline/` -> `review/` or `trash/`)
- `logs/runs/{YYYY-MM-DD}.log.md` — run log with pipeline snapshot
- `logs/changelog.md` — append-only promoted idea entries
- `logs/metrics.json` — cumulative pipeline stats
- `logs/idea-registry.md` — updated outcomes for terminal ideas

## Error Handling

| Scenario | Behavior |
|---|---|
| Subagent fails | Log error, skip idea, continue. Status unchanged. |
| Malformed notes entry | Log warning, treat as no-op. Status unchanged. |
| Constraints file missing | Abort brainstorm phase only. Process existing ideas. |
| Unrecognized frontmatter status | Log error, skip idea. |
| Claude Code usage limit mid-run | Already-processed ideas keep updated status. Unprocessed ideas wait for next run. Run log captures partial progress. |
| Folder permissions error | Abort run. Log error. |
