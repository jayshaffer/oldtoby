# Brainstorm Agent Tasks

## 1. Constraints file template
- [x] Create `config/constraints.md` with the suggested section structure: Domain Focus, Technical Boundaries, Market Preferences, Anti-Patterns, Current Interests
- [x] Add placeholder content and comments explaining each section's purpose

## 2. Brainstorm command file
- [ ] Create `.claude/commands/brainstorm.md` with the brainstorm agent prompt
- [ ] Implement context loading instructions: read `config/constraints.md` and `logs/idea-registry.md`
- [ ] Implement generation step: produce 8-10 raw sketches with working title, 2-sentence pitch, target customer, and keywords
- [ ] Implement dedup step: compare each sketch against registry entries using keyword overlap, problem similarity, and title similarity
- [ ] Implement dedup logging: append duplicate entries to registry with `Status: duplicate` and `Outcome: Deduped against {matched-slug}`
- [ ] Implement replacement generation: for each duplicate, generate replacement sketch (up to 3 retries per slot)
- [ ] Implement thematic saturation detection: if 3+ sketches match registry, flag warning for run log
- [ ] Implement self-critique step: evaluate surviving sketches on constraint alignment and specificity
- [ ] Implement selection step: rank sketches, select top 5 (configurable via config), tighten pitches based on critique
- [ ] Implement write step: create `pipeline/{slug}/one-pager.md` for each selected idea using the one-pager template format
- [ ] Implement registry update: append all ideas (selected and duplicates) to `logs/idea-registry.md`

## 3. One-pager template validation
- [ ] Verify one-pager includes all required frontmatter: Slug, Generated (ISO timestamp), Status (brainstormed), Iteration (0), Keywords
- [ ] Verify one-pager includes all required sections: Elevator Pitch, Problem Statement, Proposed Solution, Target Customer, Revenue Model, Differentiation, Open Questions
- [ ] Verify one-pager target length is 300-500 words

## 4. Testing
- [ ] Test: Run `/brainstorm` with a constraints file and empty registry, verify 5 one-pagers are created in `pipeline/`
- [ ] Test: Run `/brainstorm` with existing registry entries, verify duplicates are detected and replaced
- [ ] Test: Verify registry is updated with both selected and duplicate entries
- [ ] Test: Verify thematic saturation warning triggers when 3+ sketches match registry
- [ ] Test: Verify one-pager format matches the template spec
