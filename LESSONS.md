# LESSONS.md — Inherited Lessons

These lessons are inherited from prior skill package development and adapted for the ThatOpen context.

## L-001: Always Read SKILL.md First
Never operate from memory. Every session begins by reading the relevant SKILL.md file. LLM knowledge of APIs drifts — the skill file is the source of truth.

## L-002: Source Verification Prevents Hallucination
Without traceable sources, LLMs confidently generate plausible but incorrect API calls. Every code pattern must link to a verified source in SOURCES.md.

## L-003: Version Pinning Saves Hours
A single version mismatch (e.g., `@thatopen/components` 1.x vs 2.x) can waste an entire session debugging non-existent APIs. Pin versions explicitly.

## L-004: Batch Order Is Non-Negotiable
Attempting to write implementation skills before core skills are solid leads to rework. The dependency graph exists for a reason.

## L-005: Trigger Design Is an Art
Too broad: skill fires on unrelated queries. Too narrow: skill misses valid use cases. Include positive AND negative trigger examples.

## L-006: Error Skills Are High-Value
Developers search for error messages. Skills that match specific error strings and provide targeted fixes get the most use.

## L-007: Disposal Patterns Are Critical in 3D
Three.js and ThatOpen objects (geometries, materials, textures, fragments) must be explicitly disposed. Memory leaks in BIM viewers are the number one production issue.

## L-008: WASM Initialization Is Fragile
web-ifc requires WASM file loading. Path configuration, CDN hosting, and bundler settings are common failure points. Document these explicitly.

## L-009: IFC Schema Differences Are Subtle
IFC2x3 and IFC4 have different entity names for the same concepts (e.g., `IfcBuildingStorey` vs `IfcBuildingStorey` but with different property sets). Skills must specify which schema they target.

## L-010: Streaming Is Not Optional for Production
Any IFC model over ~50MB will cause browser performance issues without fragment streaming. This must be communicated early and clearly.

## L-011: Keep Examples Complete
Partial code snippets without imports or setup context frustrate developers. Every example should be copy-pasteable into a working project.

## L-012: Document the Happy Path and the Error Path
Skills that only show the success case leave developers stranded when things fail. Include at least one error handling pattern per skill.

## L-013: Test Against Real IFC Files
Synthetic or trivial IFC files may not exercise real-world edge cases. Validation should use production-representative models when possible.

## L-014: Community Patterns May Diverge from Official
ThatOpen community examples sometimes use deprecated patterns or workarounds. Always prefer official documentation and source code.

## L-015: Changelog Discipline Prevents Confusion
Without a changelog, it is impossible to know what changed between sessions. Every modification gets logged, no matter how small.
