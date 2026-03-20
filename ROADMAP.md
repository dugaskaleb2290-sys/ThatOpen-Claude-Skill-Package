# ROADMAP.md — ThatOpen Skill Package

## Batch Plan

### B0: Foundation — DONE
- [x] CLAUDE.md — Master config
- [x] ROADMAP.md — This file
- [x] WAY_OF_WORK.md — 7-phase methodology
- [x] REQUIREMENTS.md — Quality guarantees
- [x] DECISIONS.md — Architectural decisions
- [x] LESSONS.md — Inherited lessons
- [x] SOURCES.md — Verified sources
- [x] INDEX.md — Skill catalog
- [x] CHANGELOG.md — Change history
- [x] OPEN-QUESTIONS.md — Unresolved questions
- [x] START-PROMPT.md — Session bootstrap prompt
- [x] .gitignore
- [x] .claude/settings.local.json

### Research: Deep Research — DONE
- [x] docs/research/vooronderzoek-thatopen.md — API surface research
- [x] docs/masterplan/thatopen-masterplan.md — Definitive masterplan (18 skills, 7 batches)
- [x] D-005: Version target updated to 3.3.x (was 2.x)

### B1: Core Skills — DONE
- [x] thatopen-core-architecture — Component system, lifecycle, world concept
- [x] thatopen-core-web-ifc — WASM-based IFC parser, schema handling
- [x] thatopen-core-fragments — Fragment-based geometry, worker architecture

### B2: Syntax Skills — DONE
- [x] thatopen-syntax-components — Component API patterns, get(), lifecycle
- [x] thatopen-syntax-ifc-loading — IfcLoader, WASM config, IfcImporter
- [x] thatopen-syntax-properties — Classifier, properties, spatial structure

### B3: UI + Viewer — DONE
- [x] thatopen-syntax-ui — @thatopen/ui, ui-obc, Lit web components
- [x] thatopen-impl-viewer — World setup, renderer, camera, init/dispose
- [x] thatopen-impl-navigation — Camera modes, projection, fit, controls

### B4: Interactive Tools — DONE
- [x] thatopen-impl-highlighting — Highlighter, Hoverer, Outliner, Mesher
- [x] thatopen-impl-measurements — Length, Area, Volume, Angle
- [x] thatopen-impl-clipping-plans — Clipper, ClipStyler, View, floor plans

### B5: OpenBIM + Federation — DONE
- [x] thatopen-impl-bcf — BCF topics, viewpoints, import/export
- [x] thatopen-impl-federation — Multi-model, coordination, Hider, BoundingBoxer

### B6: Error Skills — DONE
- [x] thatopen-errors-loading — IFC parse failures, WASM init, worker errors
- [x] thatopen-errors-performance — Memory leaks, disposal, large models

### B7: Agent Skills — DONE
- [x] thatopen-agents-viewer-builder — End-to-end viewer scaffolding agent
- [x] thatopen-agents-model-analyzer — IFC model analysis and reporting agent

## Progress Summary

| Batch | Skills | Status |
|---|---|---|
| B0 | Foundation files | DONE |
| Research | Vooronderzoek + Masterplan | DONE |
| B1 | 3 core skills | DONE |
| B2 | 3 syntax skills | DONE |
| B3 | 3 UI + viewer skills | DONE |
| B4 | 3 interactive tools | DONE |
| B5 | 2 OpenBIM + federation | DONE |
| B6 | 2 error skills | DONE |
| B7 | 2 agent skills | DONE |
| **Total** | **18 skills** | |
