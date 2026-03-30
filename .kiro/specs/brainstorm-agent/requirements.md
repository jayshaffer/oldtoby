# Brainstorm Agent Requirements

## User Story 1: Constraints-Driven Idea Generation
As a product ideation operator, I want the brainstorm agent to generate ideas aligned with my constraints file,
so that the pipeline stays focused on ideas I actually care about.

### Acceptance Criteria
- WHEN invoked, THE SYSTEM SHALL read `config/constraints.md` and `logs/idea-registry.md` into context.
- WHEN generating ideas, THE SYSTEM SHALL produce 8-10 raw idea sketches (configurable via `brainstorm.raw_sketches` in config), each with a working title, 2-sentence pitch, target customer, and keywords.
- WHEN generating ideas, THE SYSTEM SHALL evaluate each sketch against the constraints file for domain focus, technical boundaries, and market preferences.
- WHEN generating ideas, THE SYSTEM SHALL NOT write raw sketches to disk — they are internal working notes only.

## User Story 2: Semantic Deduplication
As the pipeline, I want the brainstorm agent to detect duplicate ideas against the full idea history,
so that the pipeline doesn't waste cycles re-evaluating ideas that already went through the gauntlet.

### Acceptance Criteria
- WHEN checking for duplicates, THE SYSTEM SHALL compare each sketch's problem statement, keywords, and solution shape against every entry in `logs/idea-registry.md`.
- WHEN two ideas solve the same core problem for the same customer (even with different solutions or slugs), THE SYSTEM SHALL flag them as duplicates.
- WHEN a duplicate is detected, THE SYSTEM SHALL discard the sketch, append a registry entry with `Status: duplicate` and `Outcome: Deduped against {matched-slug}`, and generate a replacement sketch.
- WHEN generating replacement sketches, THE SYSTEM SHALL retry up to 3 times per slot before accepting whatever is generated.
- WHEN 3+ of the raw sketches match existing registry entries, THE SYSTEM SHALL flag a thematic saturation warning in the run log.

## User Story 3: Self-Critique and Selection
As the pipeline, I want the brainstorm agent to critique its own output before writing ideas to the pipeline,
so that only well-formed, specific ideas enter the review gauntlet.

### Acceptance Criteria
- WHEN critiquing surviving sketches, THE SYSTEM SHALL evaluate on two axes: constraint alignment and specificity (is the problem concrete enough that a PM could write a PRD?).
- WHEN selecting ideas, THE SYSTEM SHALL rank sketches and select the top 5 (configurable via `brainstorm.ideas_per_run`).
- WHEN writing selected ideas, THE SYSTEM SHALL tighten each pitch based on critique findings before producing the one-pager.

## User Story 4: One-Pager Output
As the pipeline, I want each brainstormed idea written as a structured one-pager with proper frontmatter,
so that the orchestrator can route it and agents can process it.

### Acceptance Criteria
- WHEN writing an idea, THE SYSTEM SHALL create a folder `pipeline/{idea-slug}/` containing `one-pager.md`.
- WHEN writing `one-pager.md`, THE SYSTEM SHALL include frontmatter: slug, generated timestamp, `Status: brainstormed`, `Iteration: 0`, and keywords.
- WHEN writing `one-pager.md`, THE SYSTEM SHALL include sections: Elevator Pitch, Problem Statement, Proposed Solution, Target Customer, Revenue Model, Differentiation, and Open Questions.
- WHEN writing ideas, THE SYSTEM SHALL keep one-pagers between 300-500 words.
- WHEN all ideas are written, THE SYSTEM SHALL append entries for all generated ideas (selected and duplicates) to `logs/idea-registry.md`.

## User Story 5: Registry Entries
As the pipeline, I want every idea ever generated recorded in the idea registry,
so that the brainstorm agent has institutional memory across runs.

### Acceptance Criteria
- WHEN appending to the registry, THE SYSTEM SHALL include: slug, title, problem (one-line), keywords, status, date, and outcome.
- WHEN appending duplicates, THE SYSTEM SHALL set status to `duplicate` and outcome to `Deduped against {matched-slug}`.
- WHEN appending selected ideas, THE SYSTEM SHALL set status to `brainstormed` and outcome to blank (pending).
