# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OldToby is an autonomous product ideation pipeline powered by Claude Code. It runs as a nightly batch process that brainstorms SaaS/tool ideas, routes them through a multi-agent review gauntlet (PM, Market Analyst, Tech Lead, CFO, Decision Agent), and promotes viable ideas to a human review folder. The filesystem is the state machine — no database, no external services.

**Status:** Specification complete (`spec.md`), implementation in progress. Detailed component specs live in `.kiro/specs/`.

## How It Runs

There is no build system, package manager, or test framework. The entire system is Claude Code slash commands executed via cron:

```bash
# Nightly pipeline run
claude -p "/run-pipeline" --model sonnet

# Manual send-back from review
claude -p "/send-back {idea-slug}"
```

All commands live in `.claude/commands/`. Each agent is a standalone command file.

## Architecture

**Pipeline stages** — each idea progresses through frontmatter `status` in `one-pager.md`:

```
brainstormed → prd-ready → market-ready → tech-ready → cfo-ready → promoted | killed
```

**Agent dispatch** — the orchestrator (`run-pipeline.md`) reads status, spawns the appropriate agent as a subagent:
- `brainstormed` → PM agent → produces `prd.md`
- `prd-ready` → Market agent → produces `market-analysis.md`
- `market-ready` → Tech agent → produces `solution-doc.md`
- `tech-ready` → CFO agent → produces `financial-assessment.md`
- `cfo-ready` → Decision agent → promotes, kills, or sends back

**Key directories:**
- `pipeline/` — active ideas (each in `pipeline/{slug}/`)
- `review/` — promoted ideas awaiting human review
- `trash/` — killed ideas
- `config/` — `pipeline.config.json`, `agent-profiles.md`, `constraints.md`
- `logs/` — `idea-registry.md` (append-only), `changelog.md`, `metrics.json`, `runs/`

**State lives in frontmatter.** Routing decisions are driven by the `status` and `iteration` fields in each idea's `one-pager.md`. Agents append verdicts to `notes.md` (append-only). The orchestrator reads verdicts and updates frontmatter.

## Key Design Constraints

- **Filesystem as state** — no database. Frontmatter drives all routing. Ideas are folders, status is YAML.
- **Append-only logs** — `notes.md`, `idea-registry.md`, and `changelog.md` are never truncated or rewritten.
- **Subagent isolation** — each agent runs in its own context window; the orchestrator stays lightweight by delegating.
- **Surgical send-backs** — the decision agent targets a specific agent and artifact for revision, not a full restart.
- **Circuit breaker** — ideas exceeding `max_iterations_per_idea` (default 5) must be promoted or killed, no more send-backs.
- **Idempotency** — if today's run log exists in `logs/runs/`, brainstorm is skipped.
- **Context bloat as kill signal** — if an idea folder exceeds ~8000 tokens and the config flag is set, it's a negative signal.

## Spec Structure

The main specification is `spec.md`. Component-level breakdowns are in `.kiro/specs/`:

| Spec | What it covers |
|------|---------------|
| `pipeline-orchestrator/` | Orchestrator command, phases, reconciliation, terminal moves |
| `brainstorm-agent/` | Idea generation, dedup, thematic saturation |
| `review-agents/` | PM, Market, Tech, CFO agents + refresh passes |
| `decision-and-refinement/` | Promote/kill/send-back logic, downstream cascade |
| `config-and-logging/` | Config files, run logs, changelog, metrics |

Each contains `requirements.md`, `design.md`, and `tasks.md`. The master task list is at `.kiro/specs/tasks.md`.
