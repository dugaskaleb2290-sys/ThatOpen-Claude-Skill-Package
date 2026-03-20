# Federation API Reference

## Hider

Controls element visibility at the fragment level. Operates on ModelIdMap for
per-model and cross-model visibility management.

### Class Definition

```typescript
class Hider extends Component {
  static readonly uuid: string;
  enabled: boolean;
}
```

### Methods

#### set(visible, modelIdMap?)

Show or hide elements. When called without a ModelIdMap, affects ALL elements
in ALL loaded models.

```typescript
set(visible: boolean, modelIdMap?: ModelIdMap): void
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `visible` | `boolean` | Yes | `true` to show, `false` to hide |
| `modelIdMap` | `ModelIdMap` | No | Elements to affect. Omit to affect all. |

**Behavior:**
- `set(true)` — shows ALL elements (reset visibility)
- `set(false)` — hides ALL elements
- `set(true, items)` — shows only specified items
- `set(false, items)` — hides only specified items

#### isolate(modelIdMap)

Shows ONLY the specified elements. Everything else is hidden. This is
equivalent to calling `set(false)` followed by `set(true, modelIdMap)` but
executed atomically.

```typescript
isolate(modelIdMap: ModelIdMap): void
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `modelIdMap` | `ModelIdMap` | Yes | Elements to keep visible |

**ALWAYS use `isolate()` instead of manual hide-all-then-show.** The atomic
operation prevents flickering and ensures correct state.

#### toggle(modelIdMap)

Inverts visibility of specified elements. Visible elements become hidden;
hidden elements become visible.

```typescript
toggle(modelIdMap: ModelIdMap): void
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `modelIdMap` | `ModelIdMap` | Yes | Elements to toggle |

#### getVisibilityMap(state, modelIds?)

Returns a map of elements matching the requested visibility state.

```typescript
getVisibilityMap(state: boolean, modelIds?: string[]): Map<string, Set<number>>
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `state` | `boolean` | Yes | `true` for visible items, `false` for hidden |
| `modelIds` | `string[]` | No | Filter to specific models. Omit for all models. |

**Returns:** `Map<string, Set<number>>` — model ID to set of local element IDs.

---

## BoundingBoxer

Computes axis-aligned bounding boxes for elements and models. Used for camera
fitting, spatial queries, and view orientation.

### Class Definition

```typescript
class BoundingBoxer extends Component {
  static readonly uuid: string;
  enabled: boolean;
}
```

### Methods

#### addFromModelIdMap(items)

Add specific elements to the bounding box computation.

```typescript
addFromModelIdMap(items: ModelIdMap): void
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `items` | `ModelIdMap` | Yes | Elements to include in the bounding box |

#### addFromModels(modelIds?)

Add entire models to the bounding box computation. When called without
arguments, includes ALL loaded models.

```typescript
addFromModels(modelIds?: string[]): void
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `modelIds` | `string[]` | No | Model IDs to include. Omit for all models. |

#### get()

Returns the computed axis-aligned bounding box encompassing all added items.

```typescript
get(): THREE.Box3
```

**Returns:** `THREE.Box3` — the unified bounding box.

**ALWAYS call `addFromModelIdMap()` or `addFromModels()` before `get()`.** The
bounding box is computed incrementally from added items.

#### getCenter(modelIdMap)

Returns the centroid of the bounding box of the specified items.

```typescript
getCenter(modelIdMap: ModelIdMap): THREE.Vector3
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `modelIdMap` | `ModelIdMap` | Yes | Elements to compute center for |

**Returns:** `THREE.Vector3` — center point.

#### getCameraOrientation(orientation, offsetFactor?)

Computes camera position and look-at target for standard architectural views.

```typescript
getCameraOrientation(
  orientation: "front" | "back" | "left" | "right" | "top" | "bottom",
  offsetFactor?: number
): { position: THREE.Vector3; target: THREE.Vector3 }
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `orientation` | `string` | Yes | View direction |
| `offsetFactor` | `number` | No | Distance multiplier from bounding box center. Default: 1.0. Use >1 for more padding. |

**Returns:** Object with `position` and `target` vectors for camera placement.

**ALWAYS call `addFromModels()` or `addFromModelIdMap()` before
`getCameraOrientation()`.** The orientation is computed from the current
bounding box state.

---

## FragmentsManager — Coordination API

These methods on FragmentsManager handle multi-model coordinate alignment.

### Properties

#### baseCoordinationModel

```typescript
baseCoordinationModel: string
```

The model ID of the reference model. All other models are aligned relative to
this model's coordinate system.

**ALWAYS set this before loading additional models.**

#### baseCoordinationMatrix

```typescript
baseCoordinationMatrix: THREE.Matrix4
```

The 4x4 transformation matrix of the base model. Retrieved from the first
loaded model's `coordinationMatrix` property.

**ALWAYS set this alongside `baseCoordinationModel`.**

### Methods

#### applyBaseCoordinateSystem(object, originalMatrix?)

Transforms an object from its original coordinate system into the base
coordinate system.

```typescript
applyBaseCoordinateSystem(
  object: THREE.Object3D,
  originalMatrix?: THREE.Matrix4
): THREE.Matrix4
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `object` | `THREE.Object3D` | Yes | The object to transform (typically a FragmentsModel) |
| `originalMatrix` | `THREE.Matrix4` | No | The object's original coordination matrix |

**Returns:** `THREE.Matrix4` — the transformation matrix that was applied.

**Algorithm:**
1. Compute inverse of `baseCoordinationMatrix`
2. Multiply inverse by `originalMatrix`
3. Apply result to `object.matrix`

#### disposeModel(modelId)

Removes a single model from the scene and frees its resources.

```typescript
disposeModel(modelId: string): void
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `modelId` | `string` | Yes | The model ID to dispose |

**Behavior:** Removes the model from `fragments.list`, disposes all Fragment
GPU buffers (geometry, material, textures), removes meshes from the scene
graph, and fires `onBeforeDispose`.

---

## Classifier — Model Grouping

### byModel()

Groups all loaded elements by their source model name. Creates entries in
`classifier.list` under the `"models"` key.

```typescript
byModel(): void
```

**ALWAYS call after loading models to enable per-model queries.**

### find(classificationData)

Queries elements matching classification criteria. Returns a ModelIdMap of
matching elements across all loaded models.

```typescript
find(classificationData: {
  models?: string[];
  categories?: string[];
  storeys?: string[];
}): Promise<ModelIdMap>
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `models` | `string[]` | No | Model names to filter by |
| `categories` | `string[]` | No | IFC categories (e.g., "IFCWALL") |
| `storeys` | `string[]` | No | Building storey names |

**Returns:** `ModelIdMap` matching all specified criteria (intersection).

---

## ModelIdMap Type

```typescript
type ModelIdMap = Record<string, Set<number>>;
```

- **Keys:** Model UUID strings (from `FragmentsModel.modelId`)
- **Values:** Sets of local element IDs (numbers, unique within each model)

ALWAYS use ModelIdMap for any operation targeting specific elements. NEVER use
express IDs alone — they are only unique within a single model.
