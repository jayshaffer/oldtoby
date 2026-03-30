# OldToby

An autonomous product ideation pipeline powered by Claude Code. Runs as a nightly batch process that brainstorms SaaS/tool ideas, routes them through a multi-agent review gauntlet, and promotes viable ideas to a human review folder.

No database. No external services. The filesystem is the state machine.

## How It Works

Each night, a cron job kicks off a pipeline run. Ideas are generated, then advanced one stage per run through a gauntlet of specialized AI agents:

```
brainstormed → prd-ready → market-ready → tech-ready → cfo-ready → promoted | killed
```

| Stage | Agent | Produces |
|-------|-------|----------|
| Brainstorm | Brainstorm agent | `one-pager.md` |
| Product Review | PM agent | `prd.md` |
| Market Analysis | Market Fit agent | `market-analysis.md` |
| Technical Review | Tech Lead agent | `solution-doc.md` |
| Financial Review | CFO agent | `financial-assessment.md` |
| Final Decision | Decision agent | Promote, kill, or send back |

Ideas that survive land in `review/` for a human to read. Rejected ideas go to `trash/` with a kill rationale. The decision agent can also send an idea back to a specific earlier agent for targeted revision.

## Prerequisites

- [Claude Code](https://claude.ai/code) installed and authenticated
- A cron scheduler (or run manually)

## Usage

### Nightly pipeline run (via cron)

```bash
claude -p "/run-pipeline" --model sonnet
```

### Manual send-back from human review

```bash
claude -p "/send-back {idea-slug}"
```

### Cron setup

```cron
0 2 * * * cd /path/to/oldtoby && claude -p "/run-pipeline" --model sonnet
```

Default schedule is 2:00 AM Mountain time. Configure in `config/pipeline.config.json`.

## Project Structure

```
oldtoby/
├── config/
│   ├── constraints.md            # Human-authored ideation constraints
│   ├── agent-profiles.md         # Agent role definitions and mandates
│   └── pipeline.config.json      # Schedule, thresholds, tuning knobs
├── pipeline/                     # Active ideas in progress
│   └── {idea-slug}/
│       ├── one-pager.md          # Frontmatter status drives routing
│       ├── notes.md              # Append-only agent commentary
│       └── ...                   # Artifacts added by each agent
├── review/                       # Promoted ideas awaiting human review
├── trash/                        # Killed ideas
├── archive/                      # Previously reviewed (moved by human)
├── logs/
│   ├── idea-registry.md          # Master index of every idea generated
│   ├── changelog.md              # Append-only log of promotions
│   ├── metrics.json              # Cumulative pipeline stats
│   └── runs/                     # Per-run execution logs
└── .claude/commands/             # All agent and orchestrator commands
```

## Configuration

Edit `config/pipeline.config.json` to tune:

- **`brainstorm.ideas_per_run`** — how many ideas to generate per run (default: 5)
- **`agents.max_iterations_per_idea`** — circuit breaker limit before forced promote/kill (default: 5)
- **`context.warn_total_tokens`** — token threshold that signals context bloat (default: 8000)
- **`schedule.cron`** — cron expression for scheduling

Edit `config/constraints.md` to steer what kinds of ideas get generated.

## Design Principles

- **Filesystem as state** — frontmatter YAML in each idea's `one-pager.md` drives all routing
- **Adversarial review** — agents have distinct mandates designed to kill bad ideas early
- **Human-in-the-loop** — nothing ships without landing in `review/` for a human to read
- **Refinement over rejection** — surgical send-backs to specific agents, not full restarts
- **Append-only logs** — `notes.md`, `idea-registry.md`, and `changelog.md` are never rewritten
- **Idempotent runs** — re-running on the same day skips brainstorm, still advances existing ideas
