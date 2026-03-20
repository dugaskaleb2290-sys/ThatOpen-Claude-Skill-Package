# OPEN-QUESTIONS.md — ThatOpen Skill Package

## Open Questions

### OQ-001: @thatopen/components-front Scope
**Date**: 2026-03-20
**Status**: OPEN
**Question**: How much of `@thatopen/components-front` (browser-specific components like postprocessing, marker system) should be covered? These are front-end-only and may warrant separate skills or be folded into implementation skills.
**Impact**: Affects skill count and batch planning for B3/B4.

### OQ-002: web-ifc Version Compatibility
**Date**: 2026-03-20
**Status**: OPEN
**Question**: web-ifc has frequent releases with occasional breaking changes in the WASM API. What is the minimum version we guarantee compatibility for? Current target is 0.0.57+ but this needs verification against @thatopen/components 2.x peer dependency.
**Impact**: Affects R-002 (Version Accuracy) compliance.

### OQ-003: IFC4x3 Support Maturity
**Date**: 2026-03-20
**Status**: OPEN
**Question**: How mature is ThatOpen's IFC4x3 support? IFC4x3 adds infrastructure entities (roads, bridges, railways). Should skills cover IFC4x3-specific features or focus on IFC2x3/IFC4?
**Impact**: Affects S-03 (thatopen-syntax-properties) and C-02 (thatopen-core-web-ifc).

### OQ-004: Fragment File Format Stability
**Date**: 2026-03-20
**Status**: OPEN
**Question**: Is the .frag file format stable across @thatopen/fragments versions? Can fragments generated with one version be loaded by another? This affects streaming skill guidance.
**Impact**: Affects C-03 (thatopen-core-fragments) and S-04 (thatopen-syntax-streaming).

### OQ-005: BCF 3.0 Support
**Date**: 2026-03-20
**Status**: OPEN
**Question**: Does ThatOpen support BCF 3.0 (the latest buildingSMART standard) or only BCF 2.1? This determines the scope of the BCF implementation skill.
**Impact**: Affects I-07 (thatopen-impl-bcf).
