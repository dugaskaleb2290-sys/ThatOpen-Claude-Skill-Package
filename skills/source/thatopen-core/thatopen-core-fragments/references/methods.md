# FragmentsManager API Reference

## FragmentsManager

The central component for managing fragment models. Extends `Component`,
implements `Disposable`.

### Properties

| Property | Type | Description |
|---|---|---|
| `list` | `Map<string, FragmentsModel>` | All loaded fragment models, keyed by model UUID |
| `initialized` | `boolean` | Whether `init()` has been called |
| `baseCoordinationModel` | `string` | UUID of the model used as coordination origin |
| `baseCoordinationMatrix` | `THREE.Matrix4` | The 4x4 matrix of the base coordination model |

### Methods

#### `init(workerURL: string, options?): void`

Initializes the web worker for offloaded operations. MUST be called before
any other FragmentsManager method.

- `workerURL` — URL to the `worker.mjs` file from `@thatopen/fragments`
- ALWAYS use a URL matching your installed `@thatopen/fragments` version

#### `raycast(config): Promise<Result | undefined>`

Performs a raycast against all loaded fragment models.

```typescript
config: {
  camera: THREE.Camera,       // The active camera
  mouse: THREE.Vector2,       // Normalized device coordinates (-1 to +1)
  dom: HTMLElement,            // The renderer's DOM element
  snappingClasses?: number[]   // Optional: restrict to IFC classes
}
```

Returns a `Result` object with `modelId`, `localId`, `point`, `normal`,
`distance`, or `undefined` if nothing was hit.

#### `highlight(style: MaterialDefinition, items?: ModelIdMap): Promise<void>`

Applies a highlight material to targeted items. If `items` is omitted,
highlights all loaded elements.

```typescript
type MaterialDefinition = {
  color: THREE.Color;
  opacity: number;
};
```

#### `resetHighlight(items?: ModelIdMap): Promise<void>`

Resets highlighted items to their original materials. If `items` is omitted,
resets all highlights.

#### `getData(items: ModelIdMap, config?): Promise<Record<string, ItemData[]>>`

Extracts IFC property data for the specified items. Runs in the worker thread.

Returns a record keyed by model ID, with arrays of `ItemData` objects
containing IFC property sets, type information, and relationships.

#### `getPositions(items: ModelIdMap): Promise<THREE.Vector3[]>`

Returns the center positions of the specified items as an array of
`THREE.Vector3` objects.

#### `getBBoxes(items: ModelIdMap): Promise<THREE.Box3[]>`

Returns axis-aligned bounding boxes for the specified items.

#### `guidsToModelIdMap(guids: Iterable<string>): Promise<ModelIdMap>`

Converts IFC GlobalId (GUID) strings to a ModelIdMap. Searches across all
loaded models. GUIDs not found in any model are silently skipped.

#### `modelIdMapToGuids(modelIdMap: ModelIdMap): Promise<string[]>`

Converts a ModelIdMap back to an array of IFC GlobalId strings. Useful for
exporting selections to BCF or external systems.

#### `applyBaseCoordinateSystem(object: THREE.Object3D, originalMatrix?: THREE.Matrix4): THREE.Matrix4`

Transforms a Three.js object so it aligns with the base coordination model.

- `object` — The Three.js object to transform (typically a FragmentsModel's root)
- `originalMatrix` — The object's original coordination matrix
- Returns the applied transformation matrix

#### `disposeModel(modelId: string): void`

Disposes a single model by UUID. Removes it from `list`, disposes GPU
resources (geometry, materials, textures), and cleans up worker state.
Triggers `onBeforeDispose` before cleanup.

### Events

| Event | Type | When |
|---|---|---|
| `onFragmentsLoaded` | `Event<FragmentsModel>` | After a model finishes loading |
| `onBeforeDispose` | `Event<FragmentsModel>` | Before a model is disposed |
| `onDisposed` | `Event<void>` | After FragmentsManager itself is disposed |

---

## FragmentsModel

Represents a single loaded BIM model in the fragment system.

### Key Properties

| Property | Type | Description |
|---|---|---|
| `modelId` | `string` | Unique identifier (UUID) |
| `coordinationMatrix` | `THREE.Matrix4` | Original world positioning matrix |
| `fragments` | `Fragment[]` | Array of instanced mesh fragments |
| `boundingBox` | `THREE.Box3` | Overall bounding box of the model |

A FragmentsModel is a Three.js `Object3D` that can be added to scenes
directly. Its children are the individual Fragment instanced meshes.

---

## ModelIdMap

```typescript
type ModelIdMap = Record<string, Set<number>>;
```

The universal item-targeting data structure in ThatOpen.

- **Keys**: Model UUID strings (matching `FragmentsModel.modelId`)
- **Values**: `Set<number>` of local element IDs within that model

### Usage Patterns

```typescript
// Target specific elements in one model
const items: ModelIdMap = {
  [model.modelId]: new Set([42, 43, 44])
};

// Target elements across multiple models
const multiItems: ModelIdMap = {
  [modelA.modelId]: new Set([1, 2, 3]),
  [modelB.modelId]: new Set([10, 20])
};

// Empty map (no items)
const empty: ModelIdMap = {};
```

### Where ModelIdMap is Used

| Component | Methods |
|---|---|
| FragmentsManager | `getData()`, `getPositions()`, `getBBoxes()`, `highlight()`, `resetHighlight()`, `guidsToModelIdMap()`, `modelIdMapToGuids()` |
| Hider | `set()`, `isolate()`, `toggle()` |
| Classifier | `find()` returns ModelIdMap |
| BoundingBoxer | `addFromModelIdMap()`, `getCenter()` |
| Highlighter | `highlightByID()`, `clear()`, `selection` property |

---

## MaterialDefinition

```typescript
type MaterialDefinition = {
  color: THREE.Color;
  opacity: number;
};
```

Used by `highlight()` and `Highlighter` for defining visual styles applied
to selected/highlighted elements. The opacity controls transparency (0 = fully
transparent, 1 = fully opaque).

---

## Worker Communication

The FragmentsManager worker handles:

| Operation | Main Thread Call | Worker Processing |
|---|---|---|
| Raycast | `raycast(config)` | Intersection tests against fragment BVH |
| Data extraction | `getData(items)` | FlatBuffers property deserialization |
| Position query | `getPositions(items)` | Geometry center calculation |
| Bounding boxes | `getBBoxes(items)` | AABB computation from geometry |
| GUID mapping | `guidsToModelIdMap()` | Lookup in model's GUID index |
| GUID export | `modelIdMapToGuids()` | Reverse lookup from local IDs |

All worker operations return Promises. The worker uses `postMessage` with
transferable buffers for zero-copy data transfer where possible.
