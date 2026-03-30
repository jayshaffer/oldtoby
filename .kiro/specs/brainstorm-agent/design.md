# Brainstorm Agent Design

## Architecture

The brainstorm agent is a Claude Code custom command (`.claude/commands/brainstorm.md`) invoked as a subagent by the orchestrator. The entire brainstorm step — generate, dedup, critique, select, write — runs in a single Claude Code invocation within one context window.

### Components
- **Command file**: `.claude/commands/brainstorm.md`
- **Input**: `config/constraints.md`, `logs/idea-registry.md`
- **Output**: 5 idea folders in `pipeline/`, updated `logs/idea-registry.md`

## Data Flow

```
1. Load context: constraints.md + idea-registry.md
2. Generate 8-10 raw sketches (internal, not written to disk)
3. Dedup: compare each sketch against registry (keyword + problem matching)
   - Flag duplicates, append to registry as "duplicate"
   - Generate replacement sketches (up to 3 retries per slot)
4. Critique: evaluate surviving sketches on constraint alignment + specificity
5. Select: rank and pick top 5
6. Write: produce one-pager.md in pipeline/{slug}/ for each selected idea
7. Append all ideas (selected + duplicates) to idea-registry.md
```

### Deduplication Strategy

No vector database. Semantic matching uses:
- **Keyword overlap**: Compare the sketch's keywords against registry entry keywords
- **Problem statement similarity**: Same core problem for same customer = duplicate, even if solution differs
- **Title similarity**: Catch obvious reformulations

Examples:
- "API health monitor" vs "endpoint uptime tracker" = **duplicate** (same problem, same customer)
- "API health monitor" vs "API documentation generator" = **not duplicate** (different problem)

### Thematic Saturation Detection

If 3+ of 8-10 raw sketches match existing registry entries, the agent flags this as thematic saturation. This means the constraints file is too narrow or the idea space is exhausted. The warning is included in the run log for operator visibility.

## Interfaces

### Constraints File Format (`config/constraints.md`)

Human-authored freeform markdown with suggested sections:
- Domain Focus
- Technical Boundaries
- Market Preferences
- Anti-Patterns
- Current Interests

### One-Pager Format (`pipeline/{slug}/one-pager.md`)

Frontmatter fields: Slug, Generated, Status, Iteration, Keywords
Body sections: Elevator Pitch, Problem Statement, Proposed Solution, Target Customer, Revenue Model, Differentiation, Open Questions

### Idea Registry Format (`logs/idea-registry.md`)

Append-only markdown table: Slug | Title | Problem (one-line) | Keywords | Status | Date | Outcome

## Key Design Goals

1. **Single-invocation pipeline**: Generate, dedup, critique, and write all happen in one context window. No multi-step orchestration needed for brainstorm.
2. **Institutional memory via registry**: The idea registry prevents regeneration of previously killed, promoted, or duplicated ideas without needing a database.
3. **Constraints-driven focus**: All ideation filters through the human-authored constraints file, keeping output relevant to the operator's actual interests.
