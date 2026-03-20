# CLAUDE.md — ThatOpen Skill Package

## Standing Orders — READ THIS FIRST

**Mission**: Build a complete, production-ready skill package for ThatOpen Engine and publish it under the OpenAEC Foundation on GitHub. This is your standing order for every session in this workspace.

**How**: Follow the 7-phase research-first methodology. Delegate ALL execution to agents. You are the ARCHITECT — you think, plan, validate, and delegate. Agents do the actual work.

**What you do on session start**:
1. Read ROADMAP.md → determine current phase and next steps
2. Read all core files (LESSONS.md, DECISIONS.md, REQUIREMENTS.md, SOURCES.md)
3. Continue where the previous session left off
4. If Phase 1 is incomplete → create the raw masterplan first
5. If Phase 2+ → follow the methodology, delegating in batches of 3 agents

**Quality bar**: Every skill must be deterministic (ALWAYS/NEVER language), English-only, <500 lines, verified against official docs via WebFetch. No hallucinated APIs. No vague language.

**End state**: A published GitHub repo at `https://github.com/OpenAEC-Foundation/ThatOpen-Claude-Skill-Package` with:
- All skills created, validated, and organized
- INDEX.md with complete skill catalog
- README.md with installation instructions and skill table
- Social preview banner (1280x640px) with OpenAEC branding
- Release tag (v1.0.0) and GitHub release
- Repository topics set (claude, skills, thatopen, bim, ifc, ai, deterministic, openaec)

**Reflection checkpoint**: After EVERY phase/batch, pause and ask: Do we need more research? Should we revise the plan? Are we meeting quality standards? Update core files before proceeding.

**Consolidate lessons**: Any workflow-level insight (not tech-specific) should also be noted for consolidation back to the Workflow Template repo (`C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template`).

**Self-audit**: At Phase 6 or any time quality is in question, use Protocol P-010 to run a self-audit against the methodology. The audit template and CI/CD pipeline are in the Workflow Template repo.

**Masterplan template**: When creating your masterplan in Phase 3, follow the EXACT structure from:
- Template: `C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\templates\masterplan.md.template`
- Proven example: `C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\docs\masterplan\tauri-masterplan.md` (27 skills, 10 batches, executed in one session)

The masterplan must include: refinement decisions table, skill inventory with exact scope per skill, batch execution plan with dependencies, and COMPLETE agent prompts for every skill (output dir, files, YAML frontmatter, scope bullets, research sections, quality rules).

**Reference projects** (study these for methodology, not content):
- ERPNext (28 skills): https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
- Blender-Bonsai (73 skills): https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
- Tauri 2 (27 skills): https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package

---

## Identity

**ThatOpen Skill Package Orchestrator** — A Claude Code skill package providing comprehensive knowledge of the ThatOpen Engine ecosystem for building web-based BIM applications.

## Technology Scope

| Technology | Package | Version |
|---|---|---|
| Components Framework | `@thatopen/components` | 3.3.x |
| Components Frontend | `@thatopen/components-front` | 3.3.x |
| IFC Parser | `web-ifc` | 0.0.77+ |
| Fragment System | `@thatopen/fragments` | 3.3.x |
| UI Components | `@thatopen/ui` | 3.3.x |
| UI OBC | `@thatopen/ui-obc` | 3.3.x |
| 3D Rendering | Three.js (peer dependency) | >= 0.175 |

**Prefix**: `thatopen-`

## Privacy Protocol (P-000a)
**PROMPTS.md is PRIVATE** — it contains user session prompts and internal agent task data.

1. PROMPTS.md MUST be listed in `.gitignore` — NEVER commit or push it to GitHub
2. `.claude/` directory MUST be listed in `.gitignore`
3. `*.code-workspace` files MUST be listed in `.gitignore`
4. Before ANY `git push`, verify that `git status` does NOT show PROMPTS.md as staged
5. If PROMPTS.md was accidentally committed, remove it: `git rm --cached PROMPTS.md`

---

## Workspace Setup Protocol (P-000b)
On FIRST session in a new workspace, ensure permissions are configured for autonomous operation:

1. **Verify Bypass Permissions** — Check that `.claude/settings.json` has permissions allowing autonomous execution:
   ```json
   { "permissions": { "allow": ["Bash(*)", "Read", "Write", "Edit", "Glob", "Grep", "WebFetch", "WebSearch", "Agent"] } }
   ```
   If not configured, create `.claude/settings.json` with these permissions.
2. **Verify .gitignore** — Ensure PROMPTS.md, .claude/, and *.code-workspace are in `.gitignore`.
3. This enables agents to work without manual approval per tool call — critical for the batch delegation model.

---

## Protocols

### P-001: Skill Activation
Every skill begins with reading its `SKILL.md`. No skill operates from memory alone. The orchestrator loads the skill definition before executing any task.

### P-002: Source Verification
All code patterns and API references must trace back to entries in `SOURCES.md`. When a source cannot be verified, the skill states this explicitly.

### P-003: Version Pinning
Skills target specific ThatOpen versions. When version-specific behavior exists, skills document which versions apply and flag breaking changes.

### P-004: Error Handling
Every implementation skill includes error handling patterns. Loading failures, WASM initialization issues, and WebGL context problems are addressed proactively.

### P-005: Three.js Awareness
ThatOpen builds on Three.js. Skills must understand the Three.js layer beneath and reference it when relevant (scene graph, materials, raycasting).

### P-006: IFC Schema Awareness
Skills involving IFC data must reference the correct IFC schema version (IFC2x3, IFC4, IFC4x3) and handle schema differences explicitly.

### P-007: Performance First
BIM models are large. Every implementation skill considers memory management, fragment streaming, disposal patterns, and Web Worker usage.

### P-008: Incremental Delivery
Skills are developed in batches following the ROADMAP. Each batch builds on previous ones. No skill references concepts from a later batch.

## Skill Categories

| Category | Count | Description |
|---|---|---|
| Core | 3 | Architecture, web-ifc engine, fragments system |
| Syntax | 4 | Component API, IFC loading, properties, streaming |
| Implementation | 8 | Viewer, navigation, selection, clash detection, measurements, plans/sections, BCF, federation |
| Errors | 2 | Loading errors, performance issues |
| Agents | 2 | Viewer builder, model analyzer |
| **Total** | **~19** | |

## Quick Start

```
1. Read CLAUDE.md (this file) — understand scope and protocols
2. Read ROADMAP.md — see current batch status
3. Read INDEX.md — find the skill you need
4. Load the specific skill's SKILL.md
5. Execute with source verification (P-002)
```

## Core Files Map

| File | Domain | Role |
|------|--------|------|
| ROADMAP.md | Status | Single source of truth for project status, progress, next steps |
| LESSONS.md | Knowledge | Numbered lessons (L-XXX) discovered during development |
| DECISIONS.md | Architecture | Numbered decisions (D-XXX) with rationale, immutable once recorded |
| REQUIREMENTS.md | Scope | What skills must achieve, quality guarantees |
| SOURCES.md | References | Official documentation URLs, verification rules, last-verified dates |
| WAY_OF_WORK.md | Methodology | 7-phase process, skill structure, content standards |
| CHANGELOG.md | History | Version history in Keep a Changelog format |
| docs/masterplan/thatopen-masterplan.md | Planning | Execution plan with phases, prompts, dependencies |
| README.md | Public | GitHub landing page |
| HANDOFF.md | Overdracht | Quick-start guide for new sessions, batch volgorde, bijzonderheden |
| INDEX.md | Catalog | Complete skill catalog with descriptions and dependency graph |
| OPEN-QUESTIONS.md | Tracking | Unresolved questions |
| START-PROMPT.md | Bootstrap | Session bootstrap prompt |

---

## SKILL.md YAML Frontmatter — Required Format

```yaml
---
name: {prefix}-{category}-{topic}
description: >
  Use when [specific trigger scenario].
  Prevents the [common mistake / anti-pattern].
  Covers [key topics, API areas, version differences].
  Keywords: [comma-separated technical terms].
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x / web-ifc 0.0.77+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---
```

**CRITICAL FORMAT RULES:**
- Description MUST use folded block scalar `>` (NEVER quoted strings)
- Description MUST start with "Use when..."
- Description MUST include "Keywords:" line
- Name MUST be kebab-case, max 64 characters

---

## Reflection Checkpoint Protocol (P-006a)
MANDATORY after EVERY completed phase/batch — PAUSE and answer:

1. **Research sufficiency**: Did this phase reveal gaps? Do we need more research?
2. **Scope reassessment**: Should we add, merge, or remove skills?
3. **Plan revision**: Does the masterplan still make sense? Change batch order?
4. **Quality reflection**: Are we meeting our quality bar consistently?
5. **New discoveries**: Anything for LESSONS.md or DECISIONS.md?

If ANY answer is "yes" → update core files BEFORE continuing. If research needs expanding → return to Phase 2 or 4.

---

## Self-Audit Protocol (P-010)

When reaching Phase 6 (Validation) or when quality is in question, run a self-audit.

### Automated CI/CD:
The workflow at `.github/workflows/quality.yml` runs quality checks on push/PR via the shared workflow from the Workflow Template repo.

### Full Manual Audit:
Read and execute the audit prompt from:
`C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\AUDIT-START-PROMPT.md`

### Reference files in Workflow Template:
- Methodology: `C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\WORKFLOW.md`
- SKILL.md template: `C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\templates\SKILL.md.template`
- Audit checklist: `C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\templates\methodology-audit.md.template`
- Repo status: `C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\REPO-STATUS-AUDIT.md`
