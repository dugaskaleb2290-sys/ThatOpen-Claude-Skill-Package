# DECISIONS.md — Architectural Decisions

## D-001: Component-Centric Skill Organization

**Date**: 2026-03-20
**Status**: Accepted

**Context**: ThatOpen Engine uses a component-based architecture where everything (tools, viewers, loaders) is a Component managed by a Components instance. We needed to decide how to organize skills.

**Decision**: Organize skills around the component hierarchy — core components first, then syntax for using them, then implementation patterns that compose multiple components.

**Alternatives considered**:
- Organize by IFC workflow (load, view, query, export) — rejected because it obscures the component system that developers must understand
- Organize by file type (IFC, BCF, fragments) — rejected because most features span multiple file types

**Consequences**: Developers learn the component system naturally through the skill progression. Core skills must be solid before implementation skills make sense.

## D-002: Fragment-First Approach

**Date**: 2026-03-20
**Status**: Accepted

**Context**: ThatOpen has evolved from loading raw IFC geometry directly into Three.js to using a fragment-based system where IFC models are converted to optimized fragment files for streaming and performance.

**Decision**: Skills teach the fragment-based workflow as the primary approach. Direct IFC-to-Three.js loading is documented but marked as legacy/simple-use-case-only.

**Alternatives considered**:
- Teach both approaches equally — rejected because fragment-based is the recommended production approach
- Only teach fragments — rejected because direct loading is still valid for small models and prototyping

**Consequences**: The streaming skill (B2) becomes foundational for all large-model implementation skills.

## D-003: Three.js as Implicit Dependency

**Date**: 2026-03-20
**Status**: Accepted

**Context**: ThatOpen wraps Three.js heavily but developers still need Three.js knowledge for custom rendering, materials, and advanced scene manipulation.

**Decision**: Skills reference Three.js types and concepts when relevant but do not teach Three.js fundamentals. A Three.js Skill Package is a separate concern.

**Alternatives considered**:
- Include Three.js basics in core skills — rejected because it bloats the package scope
- Ignore Three.js entirely — rejected because developers will hit Three.js APIs in practice

**Consequences**: Skills include Three.js type references (e.g., `THREE.Scene`, `THREE.Mesh`) and note when operations drop to the Three.js layer.

## D-004: Version Target @thatopen/components 2.x

**Date**: 2026-03-20
**Status**: Superseded by D-005

**Context**: ThatOpen underwent a major rewrite from the IFC.js era (web-ifc-viewer, web-ifc-three) to the @thatopen namespace with a component-based architecture.

**Decision**: Target `@thatopen/components` 2.x as the primary version. Legacy IFC.js APIs are not covered. `web-ifc` 0.0.57+ is the minimum WASM engine version.

**Alternatives considered**:
- Cover both old and new APIs — rejected because the old API is deprecated and the migration is well-documented by ThatOpen
- Target bleeding edge only — rejected because 2.x is the stable release line

**Consequences**: Skills use `@thatopen/components`, `@thatopen/components-front`, and related 2.x packages exclusively.

## D-005: Version Target Updated to 3.3.x

**Date**: 2026-03-20
**Status**: Accepted

**Context**: Deep research revealed that `@thatopen/components` 2.x is outdated. The ecosystem has moved to 3.3.x with significant API changes (Fragments 2.0 API in v3.1.0, hover events, SMAA, volume measurement in v3.2-3.3). Current versions: @thatopen/components 3.3.3, @thatopen/fragments 3.3.6, web-ifc 0.0.77, three >= 0.175.

**Decision**: Update all version targets to:
- `@thatopen/components` 3.3.x
- `@thatopen/components-front` 3.3.x
- `@thatopen/fragments` 3.3.x
- `web-ifc` 0.0.77+
- `three` >= 0.175
- `@thatopen/ui` 3.3.x
- `@thatopen/ui-obc` 3.3.x

**Alternatives considered**:
- Stay on 2.x — rejected because 2.x is no longer the current release line and the API has changed significantly
- Target only latest minor — rejected because 3.3.x is stable with multiple patch releases

**Consequences**: All skills, CLAUDE.md, and documentation must reference 3.3.x APIs. Existing ThatOpenCompany skills need verification against 3.3.x.
