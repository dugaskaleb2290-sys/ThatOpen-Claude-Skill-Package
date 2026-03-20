# Vooronderzoek ThatOpen Engine — Deep Research

Date: 2026-03-20
Sources: GitHub main branch, npm registry, docs.thatopen.com

---

## 1. Package Ecosystem Overview

| Package | Version | License | Key Peer Dependencies |
|---|---|---|---|
| `@thatopen/components` | 3.3.3 | MIT | fragments ~3.3.0, three >=0.175, web-ifc >=0.0.74, camera-controls >=3.1.2 |
| `@thatopen/components-front` | 3.3.3 | MIT | fragments ~3.3.0, three >=0.175, web-ifc >=0.0.74 |
| `@thatopen/fragments` | 3.3.6 | MIT | three >=0.175, web-ifc >=0.0.74 |
| `web-ifc` | 0.0.77 | MPL-2.0 | standalone (WASM) |
| `@thatopen/ui` | 3.3.3 | MIT | standalone (Lit-based) |
| `@thatopen/ui-obc` | 3.3.3 | MIT | components ~3.3.0, components-front ~3.3.0, fragments ~3.3.0, three >=0.175 |

**Dependency chain:**
```
@thatopen/ui-obc
  └─ @thatopen/components-front
       └─ @thatopen/components
            ├─ @thatopen/fragments (FlatBuffers, web workers)
            │    ├─ web-ifc (WASM IFC parser)
            │    └─ three (3D rendering)
            ├─ three-mesh-bvh (accelerated raycasting)
            ├─ camera-controls (orbit/pan/zoom)
            ├─ jszip (BCF import/export)
            └─ fast-xml-parser (BCF XML)
```

---

## 2. Core Architecture (@thatopen/components)

### 2.1 Components Container (Singleton Registry)
```typescript
class Components implements Disposable {
  readonly list: DataMap<string, Component>;
  enabled: boolean;
  onDisposed: Event<void>;
  onInit: Event<undefined>;
  static release: string; // version

  get<U extends Component>(Ctor: new (c: Components) => U): U;
  add(uuid: string, instance: Component): void;
  init(): void;      // starts requestAnimationFrame loop
  dispose(): void;   // disposes all components (FragmentsManager last)
}
```
- Each component type registered once by static `uuid`
- `get()` creates instance on first call (lazy singleton)
- BVH acceleration auto-patched onto `THREE.BufferGeometry` in constructor

### 2.2 Component Base Class
```typescript
abstract class Base {
  constructor(public components: Components) {}
  isDisposeable(): this is Disposable;
  isUpdateable(): this is Updateable;
  isConfigurable(): this is Configurable<any, any>;
  isResizeable(): this is Resizeable;
  isHideable(): this is Hideable;
  isSerializable(): this is Serializable<any>;
}

abstract class Component extends Base {
  abstract enabled: boolean;
  static readonly uuid: string;
}
```

### 2.3 Lifecycle Interfaces
| Interface | Methods/Properties | Purpose |
|---|---|---|
| `Disposable` | `dispose()`, `onDisposed: Event` | Cleanup resources |
| `Updateable` | `update(delta?)`, `onBeforeUpdate`, `onAfterUpdate` | Per-frame updates |
| `Configurable` | `setup(config?)`, `config`, `isSetup`, `onSetup` | Deferred init |
| `Resizeable` | `resize()`, `getSize()`, `onResize` | Viewport resizing |
| `Hideable` | `visible: boolean` | Show/hide |
| `Createable` | `create()`, `delete()`, `endCreation()`, `cancelCreation()` | Interactive creation |
| `Serializable` | `import()`, `export()` | Persistence |
| `CameraControllable` | `controls: CameraControls` | Camera access |
| `Eventable` | `eventManager: EventManager` | Event system |

### 2.4 World System
```typescript
// Worlds manager
class Worlds extends Component implements Updateable, Disposable {
  list: DataMap<string, World>;
  create<TScene, TCamera, TRenderer>(): SimpleWorld;
  delete(world: World): void;
}

// Usage
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);
```

### 2.5 OrthoPerspectiveCamera
- Dual cameras: `threePersp` (PerspectiveCamera) + `threeOrtho` (OrthographicCamera)
- Navigation modes: Orbit, FirstPerson, Plan
- `set(mode)` — switches navigation mode
- `fit(meshes, offset)` — frame objects in view
- `projection.onChanged` — event for projection changes
- Custom navigation modes via `addCustomNavigationMode(mode)`

### 2.6 Clipper
- `create(world)` — create plane at raycast hit
- `createFromNormalAndCoplanarPoint(world, normal, point)` — programmatic
- `deleteAll(types?)` — remove all planes
- Properties: `orthogonalY`, `autoScalePlanes`, `size`, `material`
- Events: `onBeforeCreate`, `onAfterCreate`, `onBeforeDrag`, `onAfterDrag`

### 2.7 Grids & Raycasters
- `Grids.create(world)` → `SimpleGrid`
- `Raycasters.get(world)` → `SimpleRaycaster` (lazy creation)
- Auto-disposed when world is disposed

### 2.8 Views (NEW in v3)
- View system for managing viewports
- Used with ClipStyler for floor plan visualization
- Links to camera and clipping planes

### 2.9 FastModelPicker (NEW in v3)
- GPU-based element picking
- Faster alternative to raycasting for large models

---

## 3. Fragment System

### 3.1 FragmentsManager
```typescript
class FragmentsManager extends Component implements Disposable {
  list: Map<string, FragmentsModel>;
  initialized: boolean;
  baseCoordinationModel: string;
  baseCoordinationMatrix: THREE.Matrix4;

  init(workerURL: string, options?): void;
  raycast({camera, mouse, dom, snappingClasses?}): Promise<Result | undefined>;
  highlight(style: MaterialDefinition, items?: ModelIdMap): Promise<void>;
  resetHighlight(items?: ModelIdMap): Promise<void>;
  getPositions(items: ModelIdMap): Promise<THREE.Vector3[]>;
  getBBoxes(items: ModelIdMap): Promise<THREE.Box3[]>;
  getData(items: ModelIdMap, config?): Promise<Record<string, ItemData[]>>;
  guidsToModelIdMap(guids: Iterable<string>): Promise<ModelIdMap>;
  modelIdMapToGuids(modelIdMap: ModelIdMap): Promise<string[]>;
  applyBaseCoordinateSystem(object, originalMatrix?): THREE.Matrix4;
}
```
Events: `onFragmentsLoaded`, `onBeforeDispose`, `onDisposed`

### 3.2 IfcLoader
```typescript
class IfcLoader extends Component implements Disposable {
  settings: IfcFragmentSettings;
  webIfc: WEBIFC.IfcAPI;

  setup(config?: Partial<IfcFragmentSettings>): Promise<void>;
  load(data: Uint8Array, coordinate: boolean, name: string, config?): Promise<FragmentsModel>;
  readIfcFile(data: Uint8Array): Promise<number>;
  cleanUp(): void;
}
```
- WASM path configuration via `setup({ autoSetWasm: false, wasm: { path, absolute } })`
- Default imports: Projects, Sites, Buildings, Materials, Properties, key relations
- Events: `onIfcStartedLoading`, `onIfcImporterInitialized`, `onSetup`

### 3.3 Hider
```typescript
class Hider extends Component {
  set(visible: boolean, modelIdMap?: ModelIdMap): void;
  isolate(modelIdMap: ModelIdMap): void;
  toggle(modelIdMap: ModelIdMap): void;
  getVisibilityMap(state: boolean, modelIds?: string[]): Map;
}
```

### 3.4 Classifier
```typescript
class Classifier extends Component implements Disposable {
  list: DataMap<string, DataMap<string, ClassificationGroupData>>;

  byIfcBuildingStorey(): void;
  byCategory(): void;
  byModel(): void;
  aggregateItems(...): void;
  aggregateItemRelations(...): void;
  find(classificationData): Promise<ModelIdMap>;
  setGroupQuery(...): void;
}
```

### 3.5 BoundingBoxer
- `addFromModelIdMap(items)` / `addFromModels(modelIds?)`
- `get()` → unified `THREE.Box3`
- `getCenter(modelIdMap)` → `THREE.Vector3`
- `getCameraOrientation(orientation, offsetFactor)` → `{position, target}`

### 3.6 ItemsFinder
- Item search/query capabilities

### 3.7 ModelIdMap
Central data structure: `Map<string, Set<number>>` mapping model IDs to local element IDs. Used across Hider, Classifier, FragmentsManager, BoundingBoxer, Highlighter.

---

## 4. Frontend Components (@thatopen/components-front)

### 4.1 PostproductionRenderer
- Extends `RendererWith2D`
- Constructor: `(components, container, parameters?)`
- Post-processing: AO, edge detection, outlines, SMAA
- `postproduction` getter for post-processing control
- `manualDefaultStyle`, `turnOffOnManualMode`, `manualModeDelay`
- Re-exports: `PostproductionAspect`, `GlossPass`, `EdgeDetectionPassMode`

### 4.2 Highlighter
```typescript
class Highlighter extends Component implements Disposable, Eventable {
  multiple: "none" | "shiftKey" | "ctrlKey";
  zoomToSelection: boolean;
  selection: { [selectionID: string]: ModelIdMap };
  styles: DataMap<string, MaterialDefinition | null>;
  autoToggle: Set<string>;
  selectable: { [name: string]: ModelIdMap };

  setup(config?: Partial<HighlighterConfig>): void;
  highlight(name, removePrevious?, zoomToSelection?, exclude?): Promise<void>;
  highlightByID(name, modelIdMap, removePrevious?, zoomToSelection?, exclude?): Promise<void>;
  updateColors(): Promise<void>;
  clear(name?, filter?): Promise<void>;
}
```
Config: `selectName`, `selectionColor`, `autoHighlightOnClick`, `world`, `selectEnabled`, `autoUpdateFragments`, `selectMaterialDefinition` (default yellow #BCF124)

### 4.3 Hoverer (NEW in v3)
- Fade animation hover effect
- `duration`, `animation`, `delay` properties
- Events: `onHoverStarted`, `onHoverEnded`

### 4.4 Outliner
- Outline post-processing via PostproductionRenderer
- `styles: DataSet<string>` — Highlighter style references
- `color`, `thickness`, `fillColor`, `fillOpacity`
- `addItems(modelIdMap)`, `removeItems(modelIdMap)`, `clean()`

### 4.5 Mesher (NEW in v3)
- Convert fragments to regular THREE.Mesh objects
- `get(modelIdMap, config?)` → meshes
- Useful for custom rendering or export

### 4.6 Marker
- 3D marker/annotation system with auto-clustering
- `create(world, element, point, isStatic?)` → marker ID
- Clustering: distance-based, `threshold` (px), `autoCluster`
- Custom cluster element factory support

---

## 5. Measurements (@thatopen/components-front)

All extend `Measurement<T, U>` base class.

### 5.1 LengthMeasurement
- Modes: `["free", "edge"]`
- Units: length units
- Snapping to LINE, POINT, FACE

### 5.2 AreaMeasurement
- Modes: `["free", "square", "face"]`
- `pickTolerance: 0.1`, `tolerance: 0.005`

### 5.3 VolumeMeasurement
- Modes: `["free"]`
- Extra event: `onPreviewInitialized`

### 5.4 AngleMeasurement
- Modes: `["free"]`
- Constants: `ARC_SEGMENTS: 32`, `ARC_RADIUS_FACTOR: 0.3`

### 5.5 Shared Measurement Features
- `list: DataSet<T>` — all measurements
- `lines`, `fills`, `labels`, `volumes` — visual elements
- `rounding`, `units`, `color`, `snapDistance`, `pickerSize`, `pickerMode`
- `Measurement.valueFormatter` — static custom formatter
- Events: `onPointerStop`, `onPointerMove`, `onStateChanged`, `onEnabledChange`

---

## 6. Plans & Sections (v3 Architecture)

**BREAKING CHANGE from v2**: `Plans` and `ClipEdges` components are gone. Replaced by:

### 6.1 ClipStyler
```typescript
class ClipStyler extends Component implements Disposable {
  styles: DataMap<string, ClipStyle>;
  list: DataMap<string, ClipEdges>; // readonly
  visible: boolean;

  create(plane: THREE.Plane, config?): ClipEdges;
  createFromView(view: OBC.View, config?): ClipEdges;
  createFromClipping(id: string, config?): ClipEdges;
}
```

### 6.2 New Floor Plan Workflow
1. Use `Clipper` to create clipping planes
2. Use `ClipStyler.createFromClipping(id)` to add edge visualization
3. OR use `View` system with `ClipStyler.createFromView(view)` for camera-linked plans
4. Camera `set("Plan")` mode for orthographic top-down view

---

## 7. OpenBIM Standards

### 7.1 BCFTopics
```typescript
class BCFTopics extends Component implements Disposable, Configurable {
  list: DataMap<string, Topic>;
  documents: DataMap;

  create(data?: Partial<BCFTopic>): Topic;
  load(data: Uint8Array): Promise<{viewpoints, topics}>;  // BCF v2.1 + v3.0
  export(topics?): Promise<Blob>;
  setup(config?): void;
}
```
Getters: `usedTypes`, `usedStatuses`, `usedPriorities`, `usedStages`, `usedUsers`, `usedLabels`

### 7.2 IDSSpecifications
- IDS (Information Delivery Specification) validation
- Exported from openbim module

### 7.3 Viewpoints
- `create(data?: Partial<BCFViewpoint>)` → Viewpoint
- `snapshots: DataMap<string, Uint8Array>`
- Links to BCFTopics for viewpoint references

---

## 8. UI Components

### 8.1 @thatopen/ui (Presentational)
- Built with Lit (Web Components)
- `BUI.Manager.init()` required before use
- `bim-` prefix for all elements
- Components: Button, Chart, Checkbox, ColorInput, ContextMenu, Dropdown, Grid, Icon, Input, Label, NumberInput, Option, Panel, PanelSection, Selector, Table, Tabs, TextInput, Toolbar, ToolbarSection, Tooltip, Viewport
- CSS custom properties for theming (`--bim-ui-*`)

### 8.2 @thatopen/ui-obc (Functional BIM UI)
- Pre-built components wired to @thatopen/components
- Examples: IFC load button, tool config table, classification panel
- Depends on both ui and components packages

---

## 9. web-ifc API (v0.0.77)

### 9.1 Initialization
```typescript
const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath("/wasm/");
await ifcApi.Init();
```

### 9.2 Key Methods
- `OpenModel(data, settings?)` → modelID
- `GetLine(modelID, expressID, flatten?, inverse?)` → entity
- `GetLineIDsWithType(modelID, type, includeInherited?)` → Vector
- `StreamAllMeshes(modelID, callback)` — memory-efficient geometry iteration
- `GetFlatMesh(modelID, expressID)` → triangulated geometry
- `GetCoordinationMatrix(modelID)` → 4x4 matrix
- `SaveModel(modelID)` → Uint8Array
- `CloseModel(modelID)` — free WASM memory

### 9.3 Properties Helper
- `ifcApi.properties.getItemProperties(modelID, id)`
- `ifcApi.properties.getPropertySets(modelID, elementID)`
- `ifcApi.properties.getSpatialStructure(modelID)`

### 9.4 LoaderSettings
```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;
  USE_FAST_BOOLS?: boolean;
  CIRCLE_SEGMENTS_LOW/MEDIUM/HIGH?: number;
  BOOL_ABORT_THRESHOLD?: number;
  MEMORY_LIMIT?: number;
}
```

---

## 10. Key Patterns & Anti-patterns

### DO:
1. Always use `components.get(ComponentClass)` — never instantiate directly
2. Configure WASM paths explicitly before loading IFC
3. Convert IFC to Fragments once, store binary, reload fragments
4. Initialize FragmentsManager with worker URL
5. Call `components.dispose()` on cleanup
6. Use `ModelIdMap` for all item targeting
7. Apply polygon offset to prevent z-fighting
8. Filter IFC classes during conversion for memory efficiency
9. Call `components.init()` to start the render loop

### DON'T:
1. Don't skip WASM setup — silent failures
2. Don't forget disposal — memory leaks crash browser tabs
3. Don't mix up components (core=everywhere) and components-front (browser-only)
4. Don't hardcode web-ifc version in WASM path — match installed version
5. Don't use deprecated web-ifc-three / web-ifc-viewer packages
6. Don't assume all IFC classes load — defaults exclude some for memory
7. Don't use Plans/ClipEdges from v2 docs — use ClipStyler + View in v3

---

## 11. API Changes v2 → v3

| v2 | v3 | Notes |
|---|---|---|
| Plans component | View + ClipStyler.createFromView() | Complete redesign |
| ClipEdges component | ClipStyler.create/createFromClipping() | Now managed by ClipStyler |
| No Hoverer | Hoverer component | New in fragments/ |
| No Mesher | Mesher component | New in fragments/ |
| No FastModelPicker | FastModelPicker | GPU-based picking |
| FragmentsGroup | FragmentsModel | Renamed/refactored |
| IfcStreamer | Possibly deprecated | 404 on docs, needs verification |
| Manual edge creation | ClipEdgesCreationConfig | Declarative config |
