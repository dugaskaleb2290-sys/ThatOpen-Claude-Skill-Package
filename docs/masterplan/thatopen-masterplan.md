# ThatOpen Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw research after vooronderzoek review.
Date: 2026-03-20

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-R01 | **Removed** `thatopen-impl-clash-detection` | No built-in clash detection component in @thatopen/components 3.3.x. Spatial queries are done via BoundingBoxer + Classifier, covered in other skills. |
| D-R02 | **Merged** `thatopen-syntax-streaming` into `thatopen-core-fragments` | IfcStreamer appears deprecated in v3.3.x. Streaming is now integral to the fragment worker architecture, not a separate concept. |
| D-R03 | **Added** `thatopen-syntax-ui` | @thatopen/ui and @thatopen/ui-obc are essential packages for building BIM interfaces. Not covered by any original skill. |
| D-R04 | **Renamed** `thatopen-impl-selection` to `thatopen-impl-highlighting` | v3 separates concerns: Highlighter (click), Hoverer (hover), Outliner (outline effect), Mesher (mesh export). "Selection" was too narrow. |
| D-R05 | **Renamed** `thatopen-impl-plans-sections` to `thatopen-impl-clipping-plans` | Plans component removed in v3. Now uses Clipper + ClipStyler + View system. Name reflects the actual API. |
| D-R06 | **Updated** all version targets from 2.x to 3.3.x | Research revealed ecosystem is at @thatopen/components 3.3.3, not 2.x. Recorded as D-005 in DECISIONS.md. |

**Result**: 19 raw skills → **18 definitive skills** (1 removal, 1 merge, 1 addition, 2 renames).

---

## Definitive Skill Inventory (18 skills)

### thatopen-core/ (3 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `thatopen-core-architecture` | Component system overview; Components container; Component base class; lifecycle interfaces; World system; package split (core vs front); event system; dependency chain | `Components`, `Component`, `Base`, `World`, `Worlds`, `Event<T>`, `DataMap`, `DataSet` | Vooronderzoek §2 | L | None |
| `thatopen-core-web-ifc` | web-ifc WASM engine; IfcAPI initialization; model loading; data queries (GetLine, GetLineIDsWithType); geometry extraction (FlatMesh, PlacedGeometry); properties helper; streaming meshes; GUID utilities; schema detection | `IfcAPI`, `OpenModel`, `GetLine`, `GetFlatMesh`, `StreamAllMeshes`, `LoaderSettings`, `properties.*` | Vooronderzoek §9 | L | None |
| `thatopen-core-fragments` | Fragment binary format (FlatBuffers); FragmentsModel; FragmentsManager; worker initialization; ModelIdMap; coordinate alignment; raycast; highlight; getData; GUID mapping; fragment-to-mesh conversion | `FragmentsManager`, `FragmentsModel`, `ModelIdMap`, `init(workerURL)`, `load()`, `getData()`, `raycast()` | Vooronderzoek §3 | L | core-architecture |

### thatopen-syntax/ (4 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `thatopen-syntax-components` | `components.get()` pattern; component registration via UUID; lifecycle interface implementation; Configurable setup pattern; Updateable frame loop; Disposable cleanup chain; Event system usage; DataMap/DataSet reactive collections | `get<U>()`, `add()`, `init()`, `dispose()`, `Event.add/remove/trigger`, `DataMap`, `DataSet` | Vooronderzoek §2.1-2.3 | M | core-architecture |
| `thatopen-syntax-ifc-loading` | IfcLoader setup and configuration; WASM path configuration; load() method; IfcFragmentSettings; default imports; IfcImporter (fragments-level); two loading approaches; WASM version matching | `IfcLoader`, `setup()`, `load()`, `IfcFragmentSettings`, `IfcImporter` | Vooronderzoek §3.2, §9.1-9.4 | M | core-web-ifc, core-fragments |
| `thatopen-syntax-properties` | Classifier (byCategory, byIfcBuildingStorey, byModel); find() queries; ItemsFinder; FragmentsManager.getData(); property extraction; spatial structure; IFC relationships; aggregateItems/aggregateItemRelations | `Classifier`, `find()`, `byCategory()`, `byIfcBuildingStorey()`, `getData()`, `ItemsFinder` | Vooronderzoek §3.4, §9.2-9.3 | M | core-fragments |
| `thatopen-syntax-ui` | @thatopen/ui initialization (BUI.Manager.init); bim-* web components; Panel, Toolbar, Table, Grid, inputs; CSS custom properties theming; @thatopen/ui-obc functional components; Lit integration patterns | `BUI.Manager.init()`, `<bim-panel>`, `<bim-toolbar>`, `<bim-table>`, `<bim-grid>`, `<bim-button>` | Vooronderzoek §8 | M | core-architecture |

### thatopen-impl/ (7 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `thatopen-impl-viewer` | Complete viewer setup; World creation with SimpleScene, SimpleRenderer, OrthoPerspectiveCamera; PostproductionRenderer; grid creation; init/dispose lifecycle; container element binding; Vite/bundler setup | `Worlds.create()`, `SimpleScene`, `SimpleRenderer`, `PostproductionRenderer`, `OrthoPerspectiveCamera`, `Grids.create()`, `components.init()` | Vooronderzoek §2.4, §4.1 | L | syntax-components |
| `thatopen-impl-navigation` | OrthoPerspectiveCamera modes (Orbit, FirstPerson, Plan); projection switching (perspective/ortho); fit() framing; custom navigation modes; camera controls; frustum settings; user input toggling | `camera.set("Orbit"/"FirstPerson"/"Plan")`, `camera.fit()`, `projection.onChanged`, `setUserInput()`, `addCustomNavigationMode()` | Vooronderzoek §2.5 | M | impl-viewer |
| `thatopen-impl-highlighting` | Highlighter setup and configuration; styles (MaterialDefinition); highlight by raycast and by ID; multi-select (shiftKey/ctrlKey); autoToggle; Hoverer animation; Outliner post-processing; Mesher fragment-to-mesh; FastModelPicker GPU picking | `Highlighter`, `Hoverer`, `Outliner`, `Mesher`, `FastModelPicker`, `highlightByID()`, `clear()` | Vooronderzoek §4.2-4.5, §2.9 | L | impl-viewer, core-fragments |
| `thatopen-impl-measurements` | LengthMeasurement (free, edge modes); AreaMeasurement (free, square, face); VolumeMeasurement; AngleMeasurement; snapping (LINE, POINT, FACE); units configuration; valueFormatter; measurement lifecycle (create/end/cancel/delete) | `LengthMeasurement`, `AreaMeasurement`, `VolumeMeasurement`, `AngleMeasurement`, `Measurement<T,U>` base | Vooronderzoek §5 | M | impl-viewer |
| `thatopen-impl-clipping-plans` | Clipper component (create, delete, drag); ClipStyler (styles, createFromView, createFromClipping); View system for floor plans; Plan camera mode; edge visualization; section fill; programmatic plane creation | `Clipper`, `ClipStyler`, `ClipEdges`, `ClipStyle`, `View`, `createFromNormalAndCoplanarPoint()` | Vooronderzoek §6, §2.6 | L | impl-viewer, impl-navigation |
| `thatopen-impl-bcf` | BCFTopics create/load/export; BCF v2.1 and v3.0 support; Viewpoints; topic configuration (types, statuses, priorities); document references; extensions; IDSSpecifications overview | `BCFTopics`, `Viewpoints`, `Topic`, `load()`, `export()`, `IDSSpecifications` | Vooronderzoek §7 | M | core-fragments |
| `thatopen-impl-federation` | Multi-model loading; coordination matrix (applyBaseCoordinateSystem); Hider (set/isolate/toggle); BoundingBoxer (fit, camera orientation); per-model visibility; cross-model queries via ModelIdMap | `Hider`, `BoundingBoxer`, `applyBaseCoordinateSystem()`, `isolate()`, `getCameraOrientation()` | Vooronderzoek §3.3, §3.5 | M | syntax-ifc-loading, core-fragments |

### thatopen-errors/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `thatopen-errors-loading` | WASM initialization failures; IFC parse errors; IfcLoader setup errors; web-ifc version mismatches; WASM path configuration; silent failures; COORDINATE_TO_ORIGIN issues; missing IFC classes; worker initialization failures | Error messages, `SetWasmPath`, `Init()`, `setup()`, `init(workerURL)` | Vooronderzoek §9.1, §10 | M | syntax-ifc-loading |
| `thatopen-errors-performance` | Memory leak patterns (undisposed geometries, materials, fragments); large model handling; worker thread usage; BVH raycasting; polygon offset z-fighting; browser tab crashes; disposal chain; FragmentsManager dispose order | `dispose()`, `disposeModel()`, `components.dispose()`, memory patterns | Vooronderzoek §10, §3.1 | M | core-fragments |

### thatopen-agents/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `thatopen-agents-viewer-builder` | End-to-end BIM viewer scaffolding; project setup with Vite; package installation; viewer initialization; IFC loading; highlighting; toolbar and panel UI; complete working app | All core + syntax + impl patterns | All sections | L | ALL other skills |
| `thatopen-agents-model-analyzer` | IFC model analysis agent; spatial structure extraction; property set enumeration; element counting by type; classification reports; model validation; data export | `Classifier`, `getData()`, `getSpatialStructure()`, `GetAllTypesOfModel()` | Vooronderzoek §3.4, §9.2-9.3 | M | ALL other skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `core-web-ifc`, `core-fragments` | 3 | None | Foundation, no deps |
| 2 | `syntax-components`, `syntax-ifc-loading`, `syntax-properties` | 3 | Batch 1 | Core syntax patterns |
| 3 | `syntax-ui`, `impl-viewer`, `impl-navigation` | 3 | Batch 1-2 | UI + viewer setup |
| 4 | `impl-highlighting`, `impl-measurements`, `impl-clipping-plans` | 3 | Batch 3 | Interactive tools |
| 5 | `impl-bcf`, `impl-federation` | 2 | Batch 1-2 | OpenBIM + multi-model |
| 6 | `errors-loading`, `errors-performance` | 2 | Batch 1-3 | Error handling |
| 7 | `agents-viewer-builder`, `agents-model-analyzer` | 2 | ALL above | Agent skills last |

**Total**: 18 skills across 7 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package
RESEARCH_FILE = C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\docs\research\vooronderzoek-thatopen.md
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\thatopen-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: thatopen-core-architecture

```
## Task: Create the thatopen-core-architecture skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-core\thatopen-core-architecture\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Components, Component, Base, World, Worlds API signatures)
3. references/examples.md (world setup, component registration, lifecycle patterns)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\thatopen-architecture\SKILL.md

### YAML Frontmatter
---
name: thatopen-core-architecture
description: >
  Use when creating a ThatOpen BIM application, understanding the component
  system, or reasoning about the ThatOpen architecture.
  Prevents direct component instantiation instead of using components.get().
  Covers Components container, Component base class, lifecycle interfaces,
  World system, package split (core vs front), Event system, DataMap/DataSet.
  Keywords: thatopen, components, world, scene, renderer, camera, lifecycle,
  disposable, updateable, configurable, bim, architecture.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Components container: singleton registry, get<U>(), add(), init(), dispose()
- Component base class: static uuid, enabled, Base class with runtime interface detection
- Lifecycle interfaces: Disposable, Updateable, Configurable, Resizeable, Hideable, Createable, Serializable
- World system: Worlds manager, SimpleWorld, SimpleScene, SimpleRenderer, BaseCamera
- Package split: @thatopen/components (core, works in Node.js + browser) vs @thatopen/components-front (browser-only)
- Event system: Event<T> class (add, remove, trigger, reset, enabled)
- DataMap and DataSet reactive collections
- Dependency chain: components → fragments → web-ifc → three

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 2: Core Architecture (lines 30-120)
- Section 2.3: Lifecycle Interfaces (lines 60-80)
- Section 2.4: World System (lines 80-100)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be verified against vooronderzoek research
- Include version annotations (e.g., "@thatopen/components 3.3.x")
- Include Critical Warnings section with NEVER rules
- ALWAYS use `import * as OBC from "@thatopen/components"` convention
```

#### Prompt: thatopen-core-web-ifc

```
## Task: Create the thatopen-core-web-ifc skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-core\thatopen-core-web-ifc\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (IfcAPI complete method signatures)
3. references/examples.md (init, load, query, geometry extraction, properties)
4. references/anti-patterns.md (WASM init failures, memory leaks)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\web-ifc-api\SKILL.md

### YAML Frontmatter
---
name: thatopen-core-web-ifc
description: >
  Use when working with web-ifc directly for IFC parsing, querying IFC
  entities, extracting geometry, or understanding the WASM engine beneath
  ThatOpen components.
  Prevents WASM initialization failures and memory leaks from unclosed models.
  Covers IfcAPI, SetWasmPath, Init, OpenModel, GetLine, GetFlatMesh,
  StreamAllMeshes, properties helper, LoaderSettings, schema detection.
  Keywords: web-ifc, wasm, ifc, parser, ifcapi, geometry, properties,
  spatial structure, express id, flatmesh.
license: MIT
compatibility: "Designed for Claude Code. Requires web-ifc 0.0.77+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- IfcAPI initialization: SetWasmPath, Init, Dispose
- Model loading: OpenModel, OpenModels, CreateModel, SaveModel, CloseModel
- Data queries: GetLine, GetLines, GetLineIDsWithType, GetAllLines, GetRawLineData
- Geometry extraction: GetFlatMesh, GetGeometry, GetVertexArray, GetIndexArray
- Geometry streaming: StreamAllMeshes, StreamAllMeshesWithTypes, StreamMeshes
- Properties helper: getItemProperties, getPropertySets, getSpatialStructure, getMaterialsProperties
- Coordination: GetCoordinationMatrix, SetGeometryTransformation
- GUID utilities: GetExpressIdFromGuid, GetGuidFromExpressId
- LoaderSettings: COORDINATE_TO_ORIGIN, USE_FAST_BOOLS, MEMORY_LIMIT
- Schema detection: GetModelSchema, GetAllTypesOfModel
- Key types: FlatMesh, PlacedGeometry, IfcGeometry, Vector<T>, RawLineData
- Common IFC type constants (IFCWALL, IFCSLAB, etc.)

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 9: web-ifc API (lines 200-260)
Also read the existing skill at:
C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\web-ifc-api\SKILL.md

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- ALWAYS call Init() before any other method
- ALWAYS call CloseModel to free WASM memory
- ALWAYS use .size() and .get(i) for Vector access, NEVER array indexing
- Include vertex format note: 6 floats per vertex [x,y,z,nx,ny,nz]
- Include Critical Warnings section
```

#### Prompt: thatopen-core-fragments

```
## Task: Create the thatopen-core-fragments skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-core\thatopen-core-fragments\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (FragmentsManager API, FragmentsModel, ModelIdMap)
3. references/examples.md (init worker, load, getData, raycast, coordinate alignment)
4. references/anti-patterns.md (missing worker init, disposal failures)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\thatopen-architecture\SKILL.md

### YAML Frontmatter
---
name: thatopen-core-fragments
description: >
  Use when working with ThatOpen's fragment system for BIM model storage,
  loading, querying, or understanding the optimized geometry pipeline.
  Prevents worker initialization failures and memory leaks from undisposed models.
  Covers FragmentsManager, FragmentsModel, ModelIdMap, worker initialization,
  coordinate alignment, raycast, getData, GUID mapping.
  Keywords: fragments, fragmentsmanager, modelidmap, worker, instanced mesh,
  flatbuffers, bim model, load, dispose, raycast.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/fragments 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Fragment concept: FlatBuffers binary format, GPU instancing, optimized BIM geometry
- FragmentsManager: init(workerURL), list, load(), disposeModel()
- Worker architecture: worker.mjs configuration, offloaded operations
- ModelIdMap: Map<string, Set<number>> — universal item targeting
- Data operations: getData(), getPositions(), getBBoxes()
- GUID mapping: guidsToModelIdMap(), modelIdMapToGuids()
- Coordinate alignment: applyBaseCoordinateSystem(), baseCoordinationMatrix
- Raycast: raycast({camera, mouse, dom})
- Highlight: highlight(style, items), resetHighlight()
- IFC-to-Fragment pipeline: convert once, store binary, reload
- Events: onFragmentsLoaded, onBeforeDispose, onDisposed
- @thatopen/fragments package: earcut, flatbuffers, pako dependencies

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 3: Fragment System (lines 120-180)
Also read:
C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\flatbuffers-fragments\SKILL.md

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- ALWAYS call FragmentsManager.init(workerURL) before any fragment operations
- ALWAYS dispose models via disposeModel() or components.dispose()
- NEVER skip coordinate alignment for multi-model scenarios
- Include Critical Warnings section
```

---

### Batch 2

#### Prompt: thatopen-syntax-components

```
## Task: Create the thatopen-syntax-components skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-syntax\thatopen-syntax-components\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (components.get(), add(), Event methods, DataMap/DataSet)
3. references/examples.md (custom component, lifecycle implementation, event patterns)
4. references/anti-patterns.md (direct instantiation, missing disposal, event leaks)

### YAML Frontmatter
---
name: thatopen-syntax-components
description: >
  Use when accessing ThatOpen components, implementing custom components,
  or working with the component lifecycle and event system.
  Prevents direct instantiation of components instead of using get().
  Covers components.get() singleton pattern, component registration, lifecycle
  interface implementation, Event system, DataMap, DataSet reactive collections.
  Keywords: components, get, singleton, uuid, lifecycle, event, datamap,
  dataset, disposable, updateable, configurable, component api.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- components.get<U>(ComponentClass) pattern: lazy singleton creation
- Component registration: static uuid, constructor calls components.add()
- Lifecycle interface implementation: Disposable, Updateable, Configurable
- Event<T> system: add(), remove(), trigger(), reset(), enabled
- DataMap<K,V>: reactive Map with onItemAdded, onBeforeDelete, onCleared
- DataSet<T>: reactive Set with same events
- Creating custom components: extend Component, define uuid, implement interfaces
- Components.init() / Components.dispose() lifecycle
- Config pattern: setup(config?), isSetup, onSetup, config property

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 2.1: Components Container (lines 30-50)
- Section 2.2: Component Base Class (lines 50-60)
- Section 2.3: Lifecycle Interfaces (lines 60-80)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- ALWAYS use components.get(X) — NEVER new X(components) directly
- ALWAYS implement Disposable if component holds resources
- Include complete Event<T> method reference
- Include DataMap/DataSet API
```

#### Prompt: thatopen-syntax-ifc-loading

```
## Task: Create the thatopen-syntax-ifc-loading skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-syntax\thatopen-syntax-ifc-loading\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (IfcLoader API, IfcFragmentSettings, IfcImporter)
3. references/examples.md (basic load, custom WASM path, IfcImporter approach)
4. references/anti-patterns.md (WASM misconfig, version mismatch, missing setup)

### YAML Frontmatter
---
name: thatopen-syntax-ifc-loading
description: >
  Use when loading IFC files into a ThatOpen viewer, configuring the IFC
  loader, or choosing between IfcLoader and IfcImporter approaches.
  Prevents WASM configuration failures and version mismatches.
  Covers IfcLoader setup, WASM path config, load() method,
  IfcFragmentSettings, IfcImporter (fragments-level), default IFC imports.
  Keywords: ifc, loading, ifcloader, wasm, setup, load, ifcimporter,
  fragments, web-ifc, model, uint8array.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x / web-ifc 0.0.77+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- IfcLoader: setup(config), load(data, coordinate, name), readIfcFile()
- IfcFragmentSettings: autoSetWasm, wasm path config, coordinate, excludedCategories
- WASM path configuration: local vs CDN, absolute vs relative, version matching
- Two loading approaches: IfcLoader (high-level) vs IfcImporter (low-level)
- Default IFC imports: which entity types are imported by default
- Fragment conversion workflow: IFC → Fragments binary → store → reload
- Events: onIfcStartedLoading, onIfcImporterInitialized, onSetup
- Common settings: COORDINATE_TO_ORIGIN, USE_FAST_BOOLS

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 3.2: IfcLoader (lines 130-155)
- Section 9: web-ifc API (lines 200-260) — LoaderSettings only

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- ALWAYS call setup() before load()
- ALWAYS match WASM path version to installed web-ifc version
- NEVER hardcode web-ifc version in WASM path without checking package.json
- Include both IfcLoader and IfcImporter examples
```

#### Prompt: thatopen-syntax-properties

```
## Task: Create the thatopen-syntax-properties skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-syntax\thatopen-syntax-properties\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Classifier API, getData config, ItemsFinder)
3. references/examples.md (classify, find, getData, spatial queries)
4. references/anti-patterns.md (wrong query patterns, missing classification)

### YAML Frontmatter
---
name: thatopen-syntax-properties
description: >
  Use when querying IFC properties, classifying model elements, or
  extracting data from loaded ThatOpen fragments.
  Prevents inefficient property queries and missing classifications.
  Covers Classifier (byCategory, byIfcBuildingStorey, byModel), find()
  queries, FragmentsManager.getData(), ItemsFinder, IFC relationships.
  Keywords: classifier, properties, ifc, category, storey, find, getdata,
  spatial structure, property set, quantity set, classification.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Classifier: byCategory(), byIfcBuildingStorey(), byModel()
- Classification queries: find(classificationData) → ModelIdMap
- Custom grouping: aggregateItems(), aggregateItemRelations()
- Group management: setGroupQuery(), addGroupItems(), removeItems()
- FragmentsManager.getData(items, config): property extraction
- ItemsFinder: search and filter items
- IFC relationships in fragment context
- Spatial structure traversal via classification

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 3.4: Classifier (lines 160-175)
- Section 3.1: FragmentsManager getData (lines 125-145)
Also read:
C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\ifc-bim-standards\SKILL.md (for IFC context)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include complete Classifier method reference
- Include getData() configuration options
- Include IFC schema context where relevant
```

---

### Batch 3

#### Prompt: thatopen-syntax-ui

```
## Task: Create the thatopen-syntax-ui skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-syntax\thatopen-syntax-ui\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (BUI.Manager, component list, CSS custom properties)
3. references/examples.md (panel layout, toolbar, table, property panel)
4. references/anti-patterns.md (missing init, wrong element names)

### YAML Frontmatter
---
name: thatopen-syntax-ui
description: >
  Use when building BIM user interfaces with ThatOpen UI components,
  creating panels, toolbars, or property displays.
  Prevents missing BUI.Manager.init() and incorrect component usage.
  Covers @thatopen/ui web components (bim-panel, bim-toolbar, bim-table,
  bim-grid, bim-button, inputs), CSS theming, @thatopen/ui-obc functional
  components, Lit web component patterns.
  Keywords: ui, panel, toolbar, table, grid, button, bim-panel, lit,
  web components, theming, css custom properties, ui-obc.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/ui 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- BUI.Manager.init() initialization requirement
- Core components: Panel, PanelSection, Toolbar, ToolbarSection, Button, Table, Grid, Viewport
- Input components: TextInput, NumberInput, Checkbox, ColorInput, Dropdown, Selector
- Display components: Label, Icon, Chart, Tabs, Tooltip, ContextMenu
- CSS custom properties: --bim-ui-* theming system
- Grid layout system: CSS grid-template via layout attribute
- @thatopen/ui-obc: functional BIM components wired to @thatopen/components
- Slot-based composition patterns
- Event handling: @click, @change, custom events

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 8: UI Components (lines 185-200)
Also read:
C:\Users\Freek Heijting\Documents\GitHub\ThatOpenCompany\skills\lit-bim-ui\SKILL.md (for Lit patterns)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- ALWAYS call BUI.Manager.init() before using any bim-* element
- Include complete component catalog table
- Include full BIM layout example (toolbar + panel + viewport)
```

#### Prompt: thatopen-impl-viewer

```
## Task: Create the thatopen-impl-viewer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-impl\thatopen-impl-viewer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (World setup methods, renderer options, camera setup)
3. references/examples.md (minimal viewer, PostproductionRenderer, full setup)
4. references/anti-patterns.md (missing init, wrong dispose order, WebGL context)

### YAML Frontmatter
---
name: thatopen-impl-viewer
description: >
  Use when setting up a ThatOpen BIM viewer from scratch, creating worlds,
  configuring renderers, or initializing the 3D viewport.
  Prevents missing init() calls and incorrect world setup order.
  Covers complete viewer setup: Worlds.create(), SimpleScene, SimpleRenderer,
  PostproductionRenderer, OrthoPerspectiveCamera, Grids, components.init(),
  container binding, dispose lifecycle.
  Keywords: viewer, world, scene, renderer, camera, grid, setup, init,
  dispose, postproduction, viewport, container, webgl.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- World creation: worlds.create<SimpleScene, OrthoPerspectiveCamera, SimpleRenderer>()
- Scene setup: SimpleScene, scene.setup(), lights, background
- Renderer: SimpleRenderer vs PostproductionRenderer, container element binding
- PostproductionRenderer: postproduction, AO, edge detection, SMAA
- Camera: OrthoPerspectiveCamera, dual perspective/ortho
- Grid: Grids.create(world)
- Initialization: components.init() to start render loop
- Disposal: components.dispose() cleanup chain
- Bundler setup: Vite configuration for WASM/workers
- Container element requirements

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 2.4: World System (lines 80-100)
- Section 4.1: PostproductionRenderer (lines 120-130)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include complete minimal viewer (copy-pasteable)
- Include PostproductionRenderer variant
- ALWAYS call components.init() after world setup
- ALWAYS call components.dispose() on cleanup
```

#### Prompt: thatopen-impl-navigation

```
## Task: Create the thatopen-impl-navigation skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-impl\thatopen-impl-navigation\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (OrthoPerspectiveCamera methods, ProjectionManager, NavigationMode)
3. references/examples.md (mode switching, fit, custom mode, controls)
4. references/anti-patterns.md (wrong mode usage, projection issues)

### YAML Frontmatter
---
name: thatopen-impl-navigation
description: >
  Use when configuring camera navigation modes, switching between orbit and
  first-person views, or implementing plan view for floor plans.
  Prevents incorrect projection usage and navigation mode conflicts.
  Covers OrthoPerspectiveCamera modes (Orbit, FirstPerson, Plan), projection
  switching, fit() framing, custom navigation modes, camera controls,
  frustum settings, user input toggling.
  Keywords: navigation, camera, orbit, first person, plan, projection,
  perspective, orthographic, fit, controls, zoom, pan, rotate.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Navigation modes: Orbit, FirstPerson, Plan — set() method
- Projection: perspective vs orthographic switching, projection.onChanged
- Fit/frame: fit(meshes, offset) — auto-frame objects
- Custom navigation: addCustomNavigationMode(mode)
- Camera controls: orbit, pan, zoom behavior per mode
- Frustum size for orthographic projection
- User input: setUserInput(active) toggle
- Dual cameras: threePersp + threeOrtho access

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 2.5: OrthoPerspectiveCamera (lines 100-115)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include mode comparison table
- Include Plan mode setup for floor plans
```

---

### Batch 4

#### Prompt: thatopen-impl-highlighting

```
## Task: Create the thatopen-impl-highlighting skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-impl\thatopen-impl-highlighting\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Highlighter, Hoverer, Outliner, Mesher, FastModelPicker APIs)
3. references/examples.md (click select, hover, outline, multi-select, custom styles)
4. references/anti-patterns.md (missing setup, style conflicts, performance issues)

### YAML Frontmatter
---
name: thatopen-impl-highlighting
description: >
  Use when implementing element selection, hover effects, outline
  visualization, or converting fragments to meshes in a ThatOpen viewer.
  Prevents missing Highlighter setup and style configuration errors.
  Covers Highlighter (click selection with styles), Hoverer (hover animation),
  Outliner (post-processing outlines), Mesher (fragment-to-mesh), FastModelPicker
  (GPU picking), multi-select, zoom-to-selection, autoToggle.
  Keywords: highlighter, selection, hover, outline, pick, highlight,
  multi-select, color, style, mesher, fast model picker.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components-front 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Highlighter: setup(config), highlight(), highlightByID(), clear(), styles, selection
- Multi-select: multiple property ("none", "shiftKey", "ctrlKey")
- Zoom: zoomToSelection, zoomFactor
- AutoToggle: styles that auto-deselect
- Selectable: per-style selectable element filtering
- Hoverer: hover animation with duration, delay, material
- Outliner: post-processing outlines via PostproductionRenderer
- Outliner properties: color, thickness, fillColor, fillOpacity, styles
- Mesher: get(modelIdMap, config) → THREE.Mesh conversion
- FastModelPicker: GPU-based element picking alternative

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 4.2: Highlighter (lines 130-150)
- Section 4.3: Hoverer (lines 150-155)
- Section 4.4: Outliner (lines 155-165)
- Section 4.5: Mesher (lines 165-170)
- Section 2.9: FastModelPicker (lines 115-120)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- ALWAYS call setup() on Highlighter before use
- Include MaterialDefinition type reference
- Include complete click-to-select workflow
```

#### Prompt: thatopen-impl-measurements

```
## Task: Create the thatopen-impl-measurements skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-impl\thatopen-impl-measurements\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Measurement base, Length/Area/Volume/Angle APIs)
3. references/examples.md (each measurement type setup and usage)
4. references/anti-patterns.md (missing dispose, snapping issues)

### YAML Frontmatter
---
name: thatopen-impl-measurements
description: >
  Use when adding measurement tools to a ThatOpen BIM viewer, including
  distance, area, volume, and angle measurements.
  Prevents measurement lifecycle errors and missing snapping configuration.
  Covers LengthMeasurement, AreaMeasurement, VolumeMeasurement,
  AngleMeasurement, Measurement base class, snapping, units, modes,
  valueFormatter, measurement lifecycle (create/end/cancel/delete).
  Keywords: measurement, length, area, volume, angle, distance, snap,
  dimension, measure, units, picker.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components-front 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Measurement<T,U> base class: list, lines, fills, labels, volumes
- LengthMeasurement: modes ["free", "edge"], distance between 2 points
- AreaMeasurement: modes ["free", "square", "face"], area calculation
- VolumeMeasurement: modes ["free"], volume calculation
- AngleMeasurement: modes ["free"], angle between 3 points
- Snapping: snappings array [LINE, POINT, FACE], snapDistance, pickerSize
- Units configuration per measurement type
- Measurement.valueFormatter: static custom formatting
- Lifecycle: create() → endCreation() → cancelCreation() → delete()
- Events: onPointerStop, onPointerMove, onStateChanged, onEnabledChange
- Visual elements: DimensionLine, MeasureFill, Mark, MeasureVolume
- Properties: rounding, color, visible, enabled, delay

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 5: Measurements (lines 170-200)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include measurement type comparison table
- Include mode descriptions for each type
- ALWAYS dispose measurements on cleanup
```

#### Prompt: thatopen-impl-clipping-plans

```
## Task: Create the thatopen-impl-clipping-plans skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-impl\thatopen-impl-clipping-plans\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Clipper, ClipStyler, ClipEdges, View APIs)
3. references/examples.md (basic clipping, floor plan, styled edges, programmatic)
4. references/anti-patterns.md (v2 patterns, missing ClipStyler)

### YAML Frontmatter
---
name: thatopen-impl-clipping-plans
description: >
  Use when adding clipping planes, floor plan views, or cross-section
  visualization to a ThatOpen BIM viewer.
  Prevents using deprecated v2 Plans/ClipEdges patterns.
  Covers Clipper (create, delete, drag), ClipStyler (styles, createFromView,
  createFromClipping), View system, Plan camera mode, edge visualization,
  programmatic plane creation.
  Keywords: clipper, clipping, plane, section, floor plan, clip edges,
  clip styler, view, cut, cross section.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Clipper: create(world), createFromNormalAndCoplanarPoint(), delete(), deleteAll()
- Clipper properties: orthogonalY, autoScalePlanes, size, material, visible
- ClipStyler: styles, create(plane), createFromView(view), createFromClipping(id)
- ClipEdges: edge visualization linked to clipping planes
- ClipStyle: named edge styles with colors
- View system: camera-linked plans with auto-sync
- Plan camera mode: OrthoPerspectiveCamera.set("Plan") for top-down
- Programmatic section creation workflow
- Floor plan workflow: Clipper + ClipStyler + Plan mode
- Events: onBeforeCreate, onAfterCreate, onBeforeDrag, onAfterDrag
- v2 → v3 migration: Plans → View+ClipStyler, ClipEdges → ClipStyler managed

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 6: Plans & Sections (lines 195-210)
- Section 2.6: Clipper (lines 105-115)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- NEVER use deprecated Plans or standalone ClipEdges from v2
- ALWAYS use ClipStyler for edge visualization in v3
- Include v2→v3 migration note
- Include complete floor plan workflow
```

---

### Batch 5

#### Prompt: thatopen-impl-bcf

```
## Task: Create the thatopen-impl-bcf skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-impl\thatopen-impl-bcf\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (BCFTopics, Viewpoints, Topic, IDSSpecifications APIs)
3. references/examples.md (create topic, import/export BCF, viewpoints)
4. references/anti-patterns.md (wrong BCF version, missing config)

### YAML Frontmatter
---
name: thatopen-impl-bcf
description: >
  Use when implementing BCF (BIM Collaboration Format) issue tracking,
  viewpoint management, or IDS validation in a ThatOpen viewer.
  Prevents BCF version mismatches and missing topic configuration.
  Covers BCFTopics (create, load, export), BCF v2.1 and v3.0 support,
  Viewpoints, topic configuration, IDSSpecifications overview.
  Keywords: bcf, topic, viewpoint, issue, collaboration, ids, export,
  import, bim collaboration format, snapshot.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- BCFTopics: create(data), load(zip), export(topics), setup(config)
- Topic properties: type, status, priority, assignedTo, labels, stage
- BCF config: version ("2.1"/"3.0"), author, types, statuses, priorities
- Viewpoints: create(data), snapshots, getSnapshotExtension()
- BCF import: load(Uint8Array) → {viewpoints, topics}
- BCF export: export() → Blob (zip file)
- Extensions: updateExtensions(), usedTypes/Statuses/Priorities getters
- Document references
- IDSSpecifications: overview of IDS validation capability
- Events: onSetup, onBCFImported, onDisposed

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 7: OpenBIM Standards (lines 210-240)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include BCF v2.1 vs v3.0 differences
- Include complete import/export workflow
```

#### Prompt: thatopen-impl-federation

```
## Task: Create the thatopen-impl-federation skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-impl\thatopen-impl-federation\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Hider, BoundingBoxer, coordination matrix APIs)
3. references/examples.md (multi-model load, isolate, visibility, bounding box)
4. references/anti-patterns.md (missing coordination, wrong isolation)

### YAML Frontmatter
---
name: thatopen-impl-federation
description: >
  Use when loading multiple IFC models, coordinating model positions,
  controlling per-model visibility, or working with multi-model scenarios.
  Prevents coordinate misalignment and visibility management errors.
  Covers multi-model loading, coordination matrix, Hider (set/isolate/toggle),
  BoundingBoxer (fit, camera orientation), per-model visibility, cross-model
  queries via ModelIdMap.
  Keywords: federation, multi-model, coordination, hider, visibility,
  isolate, bounding box, alignment, model, hide, show.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Multi-model loading: sequential IfcLoader.load() calls
- Coordination: applyBaseCoordinateSystem(), baseCoordinationMatrix
- Hider: set(visible, modelIdMap), isolate(modelIdMap), toggle(modelIdMap)
- Hider: getVisibilityMap(state, modelIds)
- BoundingBoxer: addFromModelIdMap(), addFromModels(), get() → Box3
- BoundingBoxer: getCenter(modelIdMap), getCameraOrientation(orientation, offsetFactor)
- Per-model queries: Classifier.byModel(), find() with model filter
- Cross-model ModelIdMap operations
- Model lifecycle: load → manipulate → disposeModel()

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 3.3: Hider (lines 155-165)
- Section 3.5: BoundingBoxer (lines 175-185)
- Section 3.1: FragmentsManager coordination (lines 130-145)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- ALWAYS use applyBaseCoordinateSystem for multi-model alignment
- Include complete multi-model workflow
```

---

### Batch 6

#### Prompt: thatopen-errors-loading

```
## Task: Create the thatopen-errors-loading skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-errors\thatopen-errors-loading\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (error types, error messages, fix patterns)
3. references/examples.md (WASM fix, path fix, version fix, worker fix)
4. references/anti-patterns.md (common mistakes that cause loading failures)

### YAML Frontmatter
---
name: thatopen-errors-loading
description: >
  Use when IFC loading fails, WASM initialization errors occur, or
  fragment worker initialization problems arise.
  Prevents common loading failures by documenting exact error patterns.
  Covers WASM init failures, IFC parse errors, WASM path misconfiguration,
  web-ifc version mismatches, worker initialization failures, missing IFC
  classes, silent failures, COORDINATE_TO_ORIGIN issues.
  Keywords: error, loading, wasm, ifc, parse, failed, worker, init,
  setup, path, version, mismatch, crash, fix.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x / web-ifc 0.0.77+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- WASM initialization failures: missing files, wrong path, MIME type errors
- IFC parse errors: corrupt files, unsupported schemas, boolean operation timeouts
- WASM path misconfiguration: relative vs absolute, CDN vs local, version mismatch
- web-ifc version mismatches: WASM file version ≠ npm package version
- Worker initialization failures: missing worker.mjs, wrong URL, CORS issues
- Missing IFC classes: default excludedCategories, elements not appearing
- Silent failures: setup() not called, init() not called
- COORDINATE_TO_ORIGIN: large coordinate issues, floating point precision
- SharedArrayBuffer: COOP/COEP header requirements for multi-threaded WASM
- Error recovery patterns

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 10: Key Patterns & Anti-patterns (lines 250-280)
- Section 9.1: web-ifc Initialization (lines 200-210)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include error message → fix mapping table
- Include diagnostic checklist
- Every error pattern must have a concrete fix
```

#### Prompt: thatopen-errors-performance

```
## Task: Create the thatopen-errors-performance skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-errors\thatopen-errors-performance\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (disposal methods, memory patterns, performance APIs)
3. references/examples.md (proper disposal, worker setup, memory monitoring)
4. references/anti-patterns.md (memory leaks, missing disposal, main thread blocking)

### YAML Frontmatter
---
name: thatopen-errors-performance
description: >
  Use when experiencing memory leaks, slow rendering, browser tab crashes,
  or performance issues with large BIM models.
  Prevents memory leaks from undisposed resources and main thread blocking.
  Covers disposal patterns, memory management, worker usage, BVH raycasting,
  polygon offset z-fighting, fragment optimization, large model strategies.
  Keywords: performance, memory, leak, dispose, slow, crash, large model,
  worker, optimization, z-fighting, disposal, gpu, browser.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Memory leak patterns: undisposed geometries, materials, textures, fragments
- Disposal chain: components.dispose() order (FragmentsManager last)
- Per-model disposal: FragmentsManager.disposeModel()
- Three.js disposal: geometry.dispose(), material.dispose(), texture.dispose()
- Worker thread usage: fragment operations offloaded to workers
- BVH raycasting: auto-patched, no manual setup needed
- Polygon offset: z-fighting prevention on overlapping surfaces
- Large model strategies: IFC class filtering, fragment streaming
- Browser tab crashes: memory exhaustion prevention
- Performance monitoring: DevTools memory profiling patterns
- Model size guidelines: when to use streaming vs full load

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 10: Key Patterns & Anti-patterns (lines 250-280)
- Section 3.1: FragmentsManager disposal (lines 125-145)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include disposal checklist
- Include memory budget guidelines
- ALWAYS dispose in correct order
- NEVER forget to dispose Three.js objects
```

---

### Batch 7

#### Prompt: thatopen-agents-viewer-builder

```
## Task: Create the thatopen-agents-viewer-builder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-agents\thatopen-agents-viewer-builder\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete setup checklist, package versions)
3. references/examples.md (complete working app: HTML + TypeScript + Vite config)
4. references/anti-patterns.md (common scaffolding mistakes)

### YAML Frontmatter
---
name: thatopen-agents-viewer-builder
description: >
  Use when scaffolding a complete ThatOpen BIM viewer application from
  scratch, including project setup, package installation, and working code.
  Prevents incomplete project setup and missing dependencies.
  Covers end-to-end viewer creation: Vite project setup, npm dependencies,
  HTML container, TypeScript viewer code, IFC loading, highlighting,
  toolbar and panel UI, complete working application.
  Keywords: scaffold, project, setup, viewer, create, new, app, vite,
  template, starter, boilerplate, complete.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Project scaffolding: Vite + TypeScript setup
- Package installation: exact versions for all @thatopen packages
- Vite configuration: WASM handling, COOP/COEP headers, optimizeDeps
- HTML structure: viewer container, UI layout
- TypeScript viewer: complete setup with world, renderer, camera, grid
- IFC loading integration: IfcLoader setup, file input, model loading
- Highlighting: Highlighter setup with click-to-select
- UI: toolbar with buttons, panel with properties
- Worker configuration: fragment worker setup
- Disposal: proper cleanup on unmount
- Complete working example (copy-pasteable)

### Research Sections to Read
From vooronderzoek-thatopen.md:
- ALL sections (this skill synthesizes everything)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- The complete example MUST be copy-pasteable and working
- Include exact package versions
- Include Vite config
- This is an AGENT skill — it guides code generation, not just documentation
```

#### Prompt: thatopen-agents-model-analyzer

```
## Task: Create the thatopen-agents-model-analyzer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\ThatOpen-Claude-Skill-Package\skills\source\thatopen-agents\thatopen-agents-model-analyzer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (analysis APIs, classification queries, data extraction)
3. references/examples.md (model summary, property report, element inventory)
4. references/anti-patterns.md (inefficient queries, missing classification)

### YAML Frontmatter
---
name: thatopen-agents-model-analyzer
description: >
  Use when analyzing an IFC model's contents, generating reports on element
  counts, properties, or spatial structure.
  Prevents inefficient model queries and incomplete analysis.
  Covers IFC model analysis: spatial structure extraction, property set
  enumeration, element counting by type, classification reports, model
  validation, data export, quality checks.
  Keywords: analyze, analysis, report, model, ifc, properties, count,
  spatial, structure, validate, quality, inventory.
license: MIT
compatibility: "Designed for Claude Code. Requires @thatopen/components 3.3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Model summary: element counts by IFC type, schema version, file size
- Spatial structure: project → site → building → storey → space hierarchy
- Property analysis: enumerate property sets, list properties per element
- Classification: Classifier.byCategory(), byIfcBuildingStorey() results
- Element inventory: count walls, slabs, doors, windows, etc.
- Data extraction: getData() for detailed element properties
- Model validation: check for missing properties, orphaned elements
- Report generation: structured output format
- web-ifc queries: GetAllTypesOfModel, GetLineIDsWithType for statistics
- This is an AGENT skill — it guides analysis workflows

### Research Sections to Read
From vooronderzoek-thatopen.md:
- Section 3.4: Classifier (lines 160-175)
- Section 9.2-9.3: web-ifc queries and properties (lines 215-250)

### Quality Rules
- English only, SKILL.md < 500 lines
- Use ALWAYS/NEVER language
- Include complete analysis workflow
- Include report output format template
- This is an AGENT skill — focus on guiding analysis, not just API docs
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── thatopen-core/
│   ├── thatopen-core-architecture/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── methods.md
│   │       ├── examples.md
│   │       └── anti-patterns.md
│   ├── thatopen-core-web-ifc/
│   │   ├── SKILL.md
│   │   └── references/
│   └── thatopen-core-fragments/
│       ├── SKILL.md
│       └── references/
├── thatopen-syntax/
│   ├── thatopen-syntax-components/
│   ├── thatopen-syntax-ifc-loading/
│   ├── thatopen-syntax-properties/
│   └── thatopen-syntax-ui/
├── thatopen-impl/
│   ├── thatopen-impl-viewer/
│   ├── thatopen-impl-navigation/
│   ├── thatopen-impl-highlighting/
│   ├── thatopen-impl-measurements/
│   ├── thatopen-impl-clipping-plans/
│   ├── thatopen-impl-bcf/
│   └── thatopen-impl-federation/
├── thatopen-errors/
│   ├── thatopen-errors-loading/
│   └── thatopen-errors-performance/
└── thatopen-agents/
    ├── thatopen-agents-viewer-builder/
    └── thatopen-agents-model-analyzer/
```
