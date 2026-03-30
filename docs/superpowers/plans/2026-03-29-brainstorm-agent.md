# Brainstorm Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the brainstorm agent command that generates 5 deduplicated, self-critiqued product ideas per run, writing them as one-pagers to the pipeline.

**Architecture:** Single Claude Code custom command file (`.claude/commands/brainstorm.md`) that acts as a prompt. When invoked, it instructs Claude to read constraints and registry, generate sketches, deduplicate, self-critique, select top ideas, write one-pager files, and update the registry. No code, no tests — this is a prompt engineering deliverable.

**Tech Stack:** Claude Code custom commands (markdown prompt files), filesystem operations via Claude tool use.

---

## File Structure

| File | Responsibility |
|------|---------------|
| `.claude/commands/brainstorm.md` | The brainstorm agent prompt — the entire deliverable |

This is a single-file deliverable. The command file is a prompt that instructs Claude Code how to behave when `/brainstorm` is invoked. All logic lives in the prompt instructions.

---

### Task 1: Create the commands directory and brainstorm command file

**Files:**
- Create: `.claude/commands/brainstorm.md`

This is the entire Phase 2 deliverable. The command file needs to cover all 7 steps from the design doc: context loading, generation, dedup, self-critique, selection, writing, and registry update.

- [ ] **Step 1: Create `.claude/commands/` directory**

```bash
mkdir -p .claude/commands
```

- [ ] **Step 2: Write the brainstorm command file**

Create `.claude/commands/brainstorm.md` with the full prompt. The file must instruct the agent to perform these steps in order:

**Section 1 — Role & Context Loading:**
- Define the agent's role: autonomous brainstorm agent for the OldToby ideation pipeline
- Instruct it to read `config/constraints.md` for domain focus, boundaries, preferences, anti-patterns, and current interests
- Instruct it to read `logs/idea-registry.md` for the full history of previously generated ideas
- Instruct it to read `config/pipeline.config.json` to get `brainstorm.ideas_per_run` (default 5) and `brainstorm.raw_sketches` (default 10)

**Section 2 — Generation:**
- Generate 8-10 raw idea sketches (configurable via `brainstorm.raw_sketches` in config)
- Each sketch must include: working title, 2-sentence pitch, target customer, and keywords
- All sketches must align with constraints file (domain focus, technical boundaries, market preferences)
- Sketches must avoid anything in the Anti-Patterns section
- Sketches are internal working notes — do NOT write them to disk

**Section 3 — Deduplication:**
- Compare each sketch against every entry in `logs/idea-registry.md`
- Match criteria: keyword overlap, problem statement similarity (same core problem + same customer = duplicate even if solution differs), title similarity
- For each duplicate found:
  - Discard the sketch
  - Generate a replacement sketch (up to 3 retries per slot)
  - Record the duplicate for later registry append with `Status: duplicate` and `Outcome: Deduped against {matched-slug}`
- If 3+ of the raw sketches match existing registry entries, flag a thematic saturation warning (include in output for the run log)

**Section 4 — Self-Critique:**
- Evaluate each surviving sketch on two axes:
  1. Constraint alignment — does it fit the domain focus, technical boundaries, and market preferences?
  2. Specificity — is the problem concrete enough that a PM could write a PRD from it?
- Note findings for each sketch (used in selection and pitch tightening)

**Section 5 — Selection:**
- Rank surviving sketches by critique scores
- Select the top 5 (or whatever `brainstorm.ideas_per_run` says)
- Tighten each selected idea's pitch based on critique findings

**Section 6 — Write One-Pagers:**
- For each selected idea, create `pipeline/{slug}/one-pager.md`
- The slug must be a URL-safe kebab-case identifier derived from the title
- Each one-pager must follow this exact format:

```markdown
---
slug: {slug}
generated: {ISO 8601 timestamp}
status: brainstormed
iteration: 0
keywords: [{comma-separated keywords}]
---

# {Title}

## Elevator Pitch
{2-3 sentences — the tightened pitch from critique}

## Problem Statement
{What specific problem does this solve? For whom? Why now?}

## Proposed Solution
{What does the product do? Key capabilities, not implementation details.}

## Target Customer
{Who is the primary user/buyer? Be specific — job title, company size, context.}

## Revenue Model
{How does this make money? Pricing approach, expected price point.}

## Differentiation
{Why this over alternatives? What's the moat?}

## Open Questions
{3-5 unresolved questions that the review agents should investigate}
```

- One-pagers must be 300-500 words (body only, not counting frontmatter)
- Also create an empty `pipeline/{slug}/notes.md` file for each idea (agents will append to this later)

**Section 7 — Registry Update:**
- Append entries for ALL generated ideas (both selected and duplicates) to `logs/idea-registry.md`
- Each entry is a row in the existing markdown table with these columns:
  - Slug: the idea slug
  - Title: the working title
  - Problem (one-line): one sentence describing the problem
  - Keywords: comma-separated keywords
  - Status: `brainstormed` for selected ideas, `duplicate` for duplicates
  - Date: today's date in ISO format
  - Outcome: blank for selected ideas, `Deduped against {matched-slug}` for duplicates

**Section 8 — Output Summary:**
- Print a summary of what was done: how many ideas generated, how many duplicates found, how many written to pipeline
- If thematic saturation was detected, include the warning
- List the slugs of all ideas written to `pipeline/`

- [ ] **Step 3: Verify the file was created correctly**

```bash
cat .claude/commands/brainstorm.md | head -5
```

Expected: The first few lines of the brainstorm command file showing the role definition.

- [ ] **Step 4: Commit**

```bash
git add .claude/commands/brainstorm.md
git commit -m "feat: add brainstorm agent command"
```

---

### Task 2: Manual Validation

This task is manual — run the command and verify output.

- [ ] **Step 1: Run the brainstorm command**

```bash
claude -p "/brainstorm" --model sonnet
```

- [ ] **Step 2: Verify one-pagers were created**

Check that 5 idea folders exist in `pipeline/`, each containing a `one-pager.md` and `notes.md`.

```bash
ls pipeline/*/one-pager.md
ls pipeline/*/notes.md
```

Expected: 5 one-pager.md files and 5 notes.md files.

- [ ] **Step 3: Validate one-pager format**

For each one-pager, verify:
- Frontmatter includes: slug, generated (ISO timestamp), status (brainstormed), iteration (0), keywords
- Body includes all 7 sections: Elevator Pitch, Problem Statement, Proposed Solution, Target Customer, Revenue Model, Differentiation, Open Questions
- Word count is 300-500 words (body only)

```bash
wc -w pipeline/*/one-pager.md
```

- [ ] **Step 4: Validate registry was updated**

```bash
cat logs/idea-registry.md
```

Expected: The table now has 5 entries with status `brainstormed` and blank outcomes.

- [ ] **Step 5: Run again to test dedup**

```bash
claude -p "/brainstorm" --model sonnet
```

Verify: new ideas are generated (not duplicates of the first run), and any detected duplicates appear in the registry with `Status: duplicate`.

- [ ] **Step 6: Commit validation results**

If any fixes were needed to the command file, commit them:

```bash
git add .claude/commands/brainstorm.md
git commit -m "fix: refine brainstorm command after validation"
```
