# REQUIREMENTS.md — Quality Guarantees

## Skill Quality Requirements

### R-001: Source Traceability
Every code pattern, API reference, and architectural claim in a skill must trace back to a verified source listed in `SOURCES.md`. Skills that cannot verify a claim must state: "Unverified — requires confirmation against [source]."

### R-002: Version Accuracy
Skills must specify which ThatOpen versions they target. When behavior differs between versions (e.g., `@thatopen/components` 2.x vs earlier), the skill documents both behaviors or explicitly states which version it covers.

### R-003: Runnable Examples
Code examples in skills must be complete enough to run in isolation (given the stated dependencies). No pseudo-code without labeling it as such. Import statements must be included.

### R-004: Error Coverage
Every implementation skill must address at least the most common failure modes for its domain. Loading skills cover parse errors. Viewer skills cover WebGL context loss. Streaming skills cover memory exhaustion.

### R-005: Disposal Patterns
ThatOpen components must be properly disposed to prevent memory leaks. Every skill that creates components, fragments, or Three.js objects must include disposal/cleanup patterns.

### R-006: TypeScript First
All code examples use TypeScript. Type annotations are included for non-obvious types. Generic parameters are documented when relevant.

### R-007: Three.js Layer Awareness
When a skill interacts with Three.js concepts (scene, camera, renderer, materials, raycasting), it must acknowledge the Three.js dependency and use correct Three.js types.

### R-008: IFC Schema Correctness
Skills dealing with IFC data must specify the IFC schema version (IFC2x3, IFC4, IFC4x3) and handle schema-specific entity names and property sets correctly.

### R-009: Performance Budgets
Implementation skills for large-model scenarios must include guidance on:
- Maximum recommended model size for the pattern
- When to switch to streaming/fragment-based approaches
- Memory disposal after operations complete

### R-010: Trigger Precision
Skill triggers must be specific enough to avoid false activations but broad enough to catch legitimate use cases. Each trigger includes at least one positive and one negative example.

## Package Quality Requirements

### R-011: Batch Integrity
No skill references concepts, APIs, or patterns from a batch that has not yet been completed. Forward references are prohibited.

### R-012: Index Completeness
Every skill in the package must have an entry in `INDEX.md`. The index entry must match the skill's actual `SKILL.md` content.

### R-013: Changelog Discipline
Every skill addition, modification, or deprecation gets a `CHANGELOG.md` entry with date and description.

### R-014: Decision Documentation
Architectural decisions that affect multiple skills are recorded in `DECISIONS.md` with rationale and alternatives considered.
