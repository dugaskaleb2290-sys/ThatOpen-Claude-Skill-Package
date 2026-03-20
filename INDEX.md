# INDEX.md — ThatOpen Skill Package Catalog

## Overview

18 skills across 5 categories for building web-based BIM applications with ThatOpen Engine 3.3.x.

## Core Skills (3)

| Skill | Description | Dependencies |
|-------|-------------|-------------|
| `thatopen-core-architecture` | Component system, lifecycle interfaces, World system, package split, Event system, DataMap/DataSet | None |
| `thatopen-core-web-ifc` | web-ifc WASM engine, IfcAPI, model loading, data queries, geometry extraction, properties helper | None |
| `thatopen-core-fragments` | Fragment binary format, FragmentsManager, workers, ModelIdMap, coordinate alignment, raycast | core-architecture |

## Syntax Skills (4)

| Skill | Description | Dependencies |
|-------|-------------|-------------|
| `thatopen-syntax-components` | components.get() singleton pattern, lifecycle implementation, Event system, DataMap/DataSet, custom components | core-architecture |
| `thatopen-syntax-ifc-loading` | IfcLoader setup, WASM path config, load(), IfcImporter, fragment conversion workflow | core-web-ifc, core-fragments |
| `thatopen-syntax-properties` | Classifier (byCategory, byStorey, byModel), find() queries, getData(), ItemsFinder, IFC relationships | core-fragments |
| `thatopen-syntax-ui` | @thatopen/ui web components (bim-panel, bim-toolbar, bim-table), CSS theming, @thatopen/ui-obc | core-architecture |

## Implementation Skills (7)

| Skill | Description | Dependencies |
|-------|-------------|-------------|
| `thatopen-impl-viewer` | World creation, SimpleScene, SimpleRenderer, PostproductionRenderer, camera, grid, init/dispose | syntax-components |
| `thatopen-impl-navigation` | OrthoPerspectiveCamera modes (Orbit, FirstPerson, Plan), projection, fit, custom modes | impl-viewer |
| `thatopen-impl-highlighting` | Highlighter, Hoverer, Outliner, Mesher, FastModelPicker, multi-select, styles | impl-viewer, core-fragments |
| `thatopen-impl-measurements` | LengthMeasurement, AreaMeasurement, VolumeMeasurement, AngleMeasurement, snapping, units | impl-viewer |
| `thatopen-impl-clipping-plans` | Clipper, ClipStyler, View system, floor plans, sections, edge visualization | impl-viewer, impl-navigation |
| `thatopen-impl-bcf` | BCFTopics, Viewpoints, BCF v2.1/v3.0 import/export, IDSSpecifications | core-fragments |
| `thatopen-impl-federation` | Multi-model loading, coordination matrix, Hider, BoundingBoxer, per-model visibility | syntax-ifc-loading, core-fragments |

## Error Skills (2)

| Skill | Description | Dependencies |
|-------|-------------|-------------|
| `thatopen-errors-loading` | WASM init failures, IFC parse errors, path misconfig, version mismatches, worker errors | syntax-ifc-loading |
| `thatopen-errors-performance` | Memory leaks, disposal patterns, large model strategies, z-fighting, worker usage | core-fragments |

## Agent Skills (2)

| Skill | Description | Dependencies |
|-------|-------------|-------------|
| `thatopen-agents-viewer-builder` | End-to-end BIM viewer scaffolding: Vite setup, packages, viewer code, IFC loading, UI | ALL other skills |
| `thatopen-agents-model-analyzer` | IFC model analysis: spatial structure, property enumeration, element inventory, validation | ALL other skills |

## Dependency Graph

```
                    ┌─────────────────────┐
                    │   core-architecture  │
                    └──────┬──────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
    ┌─────────▼──┐  ┌──────▼─────┐  ┌──▼────────────┐
    │ core-web-ifc│  │core-fragments│  │ syntax-ui     │
    └──────┬──────┘  └──────┬──────┘  └───────────────┘
           │                │
    ┌──────▼────────────────▼──────┐
    │    syntax-ifc-loading        │
    └──────┬───────────────────────┘
           │
    ┌──────▼──────┐  ┌────────────────┐
    │syntax-props  │  │syntax-components│
    └─────────────┘  └───────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   impl-viewer   │
                    └────┬───┬───┬────┘
                         │   │   │
            ┌────────────┘   │   └────────────┐
            │                │                │
    ┌───────▼──────┐ ┌──────▼───────┐ ┌──────▼──────────┐
    │impl-navigation│ │impl-highlight│ │impl-measurements│
    └───────┬──────┘ └──────────────┘ └─────────────────┘
            │
    ┌───────▼──────────┐
    │impl-clipping-plans│
    └──────────────────┘

    ┌──────────────┐  ┌───────────────┐
    │  impl-bcf    │  │impl-federation│
    └──────────────┘  └───────────────┘

    ┌──────────────┐  ┌────────────────────┐
    │errors-loading│  │errors-performance  │
    └──────────────┘  └────────────────────┘

    ┌────────────────────────┐  ┌──────────────────────────┐
    │agents-viewer-builder   │  │agents-model-analyzer     │
    └────────────────────────┘  └──────────────────────────┘
```

## Statistics

- **Total skills**: 18
- **Total files**: 72 (18 SKILL.md + 54 reference files)
- **Categories**: core (3), syntax (4), impl (7), errors (2), agents (2)
- **Target versions**: @thatopen/components 3.3.x, web-ifc 0.0.77+, Three.js >= 0.175
