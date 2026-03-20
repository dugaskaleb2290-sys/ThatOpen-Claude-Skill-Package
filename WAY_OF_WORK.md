# WAY_OF_WORK.md — 7-Phase Methodology

## Overview

Every skill in this package follows a strict 7-phase development methodology. This ensures consistency, traceability, and quality across all skills.

## The 7 Phases

### Phase 1: Research
- Study the official ThatOpen documentation and source code
- Identify the relevant API surface for this skill
- Map dependencies on other skills and packages
- Document findings in the skill's research notes

### Phase 2: Source Collection
- Gather verified code examples from ThatOpen repos
- Identify relevant GitHub issues and discussions
- Add all sources to `SOURCES.md` with proper categorization
- Every pattern must have a traceable source (Protocol P-002)

### Phase 3: Architecture
- Define the skill's scope and boundaries
- Identify input triggers (when should this skill activate?)
- Map relationships to other skills in the package
- Document decisions in `DECISIONS.md` if architectural

### Phase 4: Skill Writing
- Write the `SKILL.md` file following the standard template
- Include: description, triggers, context, instructions, examples
- Add error handling patterns (Protocol P-004)
- Include version-specific notes (Protocol P-003)

### Phase 5: Validation
- Test skill instructions against real ThatOpen code
- Verify all API references are current for target versions
- Check that examples compile and produce expected results
- Validate against `REQUIREMENTS.md` quality criteria

### Phase 6: Integration
- Ensure skill works with other skills in the same batch
- Update `INDEX.md` with the new skill entry
- Update `ROADMAP.md` status
- Add entry to `CHANGELOG.md`

### Phase 7: Review
- Cross-check all protocols (P-001 through P-008)
- Verify source traceability for every code pattern
- Confirm no forward references to later batches (P-008)
- Final quality gate before marking skill as DONE

## Phase Checklist Template

When developing a skill, use this checklist:

```
- [ ] Phase 1: Research complete, API surface mapped
- [ ] Phase 2: Sources collected and added to SOURCES.md
- [ ] Phase 3: Architecture defined, scope clear
- [ ] Phase 4: SKILL.md written with all sections
- [ ] Phase 5: Validated against real code and versions
- [ ] Phase 6: Integrated with INDEX, ROADMAP, CHANGELOG
- [ ] Phase 7: All protocols verified, quality gate passed
```

## Rules

1. **No phase skipping** — Each phase must complete before the next begins
2. **No skill from memory** — All content traces to verified sources
3. **Batch order matters** — Skills only reference completed batches
4. **Document everything** — Decisions, lessons, and open questions are captured
