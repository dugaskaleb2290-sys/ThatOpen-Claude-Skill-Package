# Measurement Methods Reference

## Measurement Base Class

### Constructor

```typescript
constructor(components: OBC.Components, measureType: U)
```

Initializes DataSets (list, lines, fills, labels, volumes) with disposal
callbacks. Registers the component via its static `uuid`.

### Abstract Members

| Member | Type | Description |
|---|---|---|
| `modes` | `string[]` | Available modes for this measurement type |
| `mode` | `string` (get/set) | Current active mode |

### Lifecycle Methods (Createable)

```typescript
create(_input?: any): void
```
Override in subclasses. Called on each user click during measurement creation.

```typescript
endCreation(_data?: T): void
```
Override in subclasses. Finalizes the current in-progress measurement and
adds it to the `list` DataSet.

```typescript
cancelCreation(): void
```
Override in subclasses. Discards the current in-progress measurement and
cleans up preview elements.

```typescript
delete(_data?: any): void
```
Override in subclasses. Removes a measurement element, typically by
raycasting against bounding boxes to find the element under the cursor.

### Disposal

```typescript
dispose(): void
```
Clears vertex picker, all DataSets (list, lines, fills, labels, volumes),
and disposes all materials (linesMaterial, fillsMaterial, volumesMaterial).

### Visual Element Factory Methods

```typescript
protected createLineElement(line: Line, startNormal?: THREE.Vector3): DimensionLine
```
Creates a DimensionLine with the current linesMaterial, linesEndpointElement,
units, rounding, and color settings.

```typescript
protected createFillElement(area: Area): MeasureFill
```
Creates a MeasureFill with the current fillsMaterial, units, and rounding.

```typescript
protected createVolumeElement(volume: Volume): MeasureVolume
```
Creates a MeasureVolume with the current volumesMaterial, units, and rounding.

```typescript
protected addLineElementsFromPoints(points: THREE.Vector3[]): DimensionLine[]
```
Generates dimension lines connecting sequential points. Returns array of
created DimensionLine instances.

### Bounding Box Retrieval

```typescript
protected getLineBoxes(): THREE.Mesh[]
```
Returns bounding box meshes from all DimensionLine elements. Used by
`delete()` for raycasting against measurements.

```typescript
protected getFillBoxes(): THREE.Mesh[]
```
Returns Three.js meshes from all MeasureFill elements.

```typescript
protected async getVolumeBoxes(): Promise<THREE.Mesh[]>
```
Async. Returns mesh arrays from all MeasureVolume elements.

### Clipping Integration

```typescript
applyPlanesVisibility(planes: THREE.Plane[]): void
```
Updates clipping planes for all visual elements (lines, fills, volumes).

### Enabled State

```typescript
get enabled(): boolean
set enabled(value: boolean)
```
When set to `true`: activates the vertex picker, registers pointer and
keyboard event listeners, fires `onEnabledChange`.
When set to `false`: deactivates the vertex picker, removes event listeners.

### Visibility

```typescript
get visible(): boolean
set visible(value: boolean)
```
When set: updates visibility for all lines, fills, and volumes. Fires
`onVisibilityChange` and `onStateChanged` with `"visibility"`.

### Units

```typescript
get units(): MeasureToUnitMap[U]
set units(value: MeasureToUnitMap[U])
```
When set: propagates units to all existing measurement items and visual
elements, converts values to the new unit. Fires `onStateChanged` with
`"units"`.

### Rounding

```typescript
get rounding(): number
set rounding(value: number)
```
When set: applies rounding to all existing measurement items and visual
elements. Fires `onStateChanged` with `"rounding"`.

### Color

```typescript
get color(): THREE.Color
set color(value: THREE.Color)
```
When set: updates all material colors (linesMaterial, fillsMaterial,
volumesMaterial) and all individual element colors.

### Material Properties

```typescript
get linesMaterial(): THREE.LineBasicMaterial
set linesMaterial(value: THREE.LineBasicMaterial)

get fillsMaterial(): THREE.MeshLambertMaterial
set fillsMaterial(value: THREE.MeshLambertMaterial)

get volumesMaterial(): THREE.MeshLambertMaterial
set volumesMaterial(value: THREE.MeshLambertMaterial)
```
Each setter disposes the previous material before assigning the new one.

### Endpoint Element

```typescript
get linesEndpointElement(): HTMLElement
set linesEndpointElement(value: HTMLElement)
```
Custom HTML element used as the endpoint marker on DimensionLines.
Default: blue rounded div created by `newDimensionMark()`.

### Vertex Picker Properties

```typescript
get pickerMode(): GraphicVertexPickerMode
set pickerMode(value: GraphicVertexPickerMode)

get snapDistance(): number
set snapDistance(value: number)

get pickerSize(): number
set pickerSize(value: number)
```
Delegates to the internal `GraphicVertexPicker` instance.

### Static Members

```typescript
static valueFormatter: ((value: number) => string) | null
```
When set, all measurement labels use this function to format their numeric
values. Applies to ALL measurement instances globally.

### unitsList Getter

```typescript
get unitsList(): string[]
```
Returns the available units for the measurement type:
- Length: `["mm", "cm", "m", "km"]`
- Area: `["mm2", "cm2", "m2", "km2"]`
- Volume: `["mm3", "cm3", "m3", "km3"]`
- Angle: `["deg", "rad"]`

---

## LengthMeasurement

**Extends**: `Measurement<Line, "length">`
**UUID**: `"2f9bcacf-18a9-4be6-a293-e898eae64ea1"`

### Properties

| Property | Type | Description |
|---|---|---|
| `modes` | `["free", "edge"]` | Available measurement modes |
| `mode` | `string` (get/set) | Current mode. Setter triggers preview reinit for "edge" |
| `isDragging` | `boolean` (getter) | Whether measurement creation is in progress |

### Methods

```typescript
create(): void
```
First call: initializes preview at the first picked point (`initPreview()`).
Second call: places endpoint and calls `endCreation()`.

```typescript
endCreation(): void
```
Finalizes the DimensionLine and adds it to the `list` DataSet.

```typescript
cancelCreation(): void
```
Discards the in-progress preview line.

```typescript
delete(): void
```
Removes the DimensionLine under the cursor by raycasting against
`getLineBoxes()`.

### Private Methods

- `initPreview()`: Establishes the first measurement point with snapping.
- `updatePreviewLine()`: Updates the preview endpoint as cursor moves.

---

## AreaMeasurement

**Extends**: `Measurement<Area, "area">`
**UUID**: `"09b78c1f-0ff1-4630-a818-ceda3d878c75"`

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `modes` | `["free", "square", "face"]` | — | Available measurement modes |
| `mode` | `string` (get/set) | `"free"` | Current mode |
| `pickTolerance` | `number` | `0.1` | Precision for selecting measurement areas |
| `tolerance` | `number` | `0.005` | Margin for point inclusion in area elements |

### Methods

```typescript
async create(): Promise<void>
```
Adds a point to the current area polygon via vertex picker. In "face" mode,
selects an entire face. In "square" mode, two clicks define a rectangle.

```typescript
endCreation(): void
```
Finalizes the area measurement. Requires minimum 3 points in "free" mode.
Creates MeasureFill and perimeter DimensionLines.

```typescript
cancelCreation(): void
```
Discards in-progress area measurement and preview elements.

```typescript
delete(): void
```
Removes the area measurement under the cursor by raycasting against
`getFillBoxes()`.

### Private Methods

- `computeLineElements()`: Generates DimensionLines between polygon points.
- `updatePreview()`: Updates the temporary fill preview during creation.

---

## VolumeMeasurement

**Extends**: `Measurement<Volume, "volume">`
**UUID**: `"01f885ab-ec4e-4e6c-a853-9dfc0d6766ed"`

### Properties

| Property | Type | Description |
|---|---|---|
| `modes` | `["free"]` | Available measurement modes |
| `mode` | `string` (get/set) | Current mode. Setter cancels creation and fires state change |

### Events

| Event | Payload | Description |
|---|---|---|
| `onPreviewInitialized` | `MeasureVolume` | Fired when preview volume object is created |

### Methods

```typescript
async initPreview(): Promise<void>
```
Creates a MeasureVolume preview instance. Fires `onPreviewInitialized`.

```typescript
async create(): Promise<void>
```
Retrieves vertex pick results and adds selected geometry items to the
preview volume.

```typescript
endCreation(): void
```
Clones the preview volume into the `list` DataSet.

```typescript
cancelCreation(): void
```
Disposes the preview volume without saving.

```typescript
delete(): void
```
Removes volume measurements under the cursor by raycasting against
`getVolumeBoxes()`.

---

## AngleMeasurement

**Extends**: `Measurement<Angle, "angle">`
**UUID**: `"2c88a142-2378-422e-b26a-bb2710841813"`

### Properties

| Property | Type | Description |
|---|---|---|
| `modes` | `["free"]` | Available measurement modes |
| `mode` | `string` (get/set) | Current mode |

### Static Constants

| Constant | Value | Description |
|---|---|---|
| `ARC_SEGMENTS` | `32` | Number of segments in the arc geometry |
| `ARC_RADIUS_FACTOR` | `0.3` | Arc radius relative to shortest arm length |
| `LABEL_OFFSET_FACTOR` | `1.4` | Label position offset from arc center |

### Methods

```typescript
async create(): Promise<void>
```
Requires three successive calls:
1. First click: sets the first point
2. Second click: sets the vertex (angle center)
3. Third click: sets the third point and calls `endCreation()`

```typescript
endCreation(): void
```
Finalizes angle after the third point. Creates arc visualization with
label showing the angle value.

```typescript
cancelCreation(): void
```
Cancels in-progress angle and disposes preview visuals.

```typescript
delete(): void
```
Removes angle measurement under the cursor via raycasting.

```typescript
dispose(): void
```
Cleans up all angle visuals (arcs, lines, labels) and base class resources.

### Private Methods

- `createAngleVisual()`: Generates complete visual with lines, arc,
  endpoints, and label.
- `updateAngleVisual()`: Updates existing visual geometry and positions.
- `createArcGeometry()` (static): Generates arc BufferGeometry between
  three points.
- `getArcMidpoint()` (static): Calculates label position on the arc.

---

## GraphicVertexPicker

The internal vertex picker used by all measurement classes.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `marker` | `Mark \| null` | — | Visual snap indicator |
| `world` | `OBC.World \| null` | — | World reference for raycasting |
| `mode` | `GraphicVertexPickerMode` | `DEFAULT` | Picking behavior mode |
| `maxDistance` | `number` | — | Snap threshold distance (world units) |
| `pickerSize` | `number` | `6` | Marker size in pixels |
| `enabled` | `boolean` | `false` | Activates/deactivates the picker |

### GraphicVertexPickerMode

| Mode | Behavior |
|---|---|
| `DEFAULT` | Uses castRay with snapping classes (LINE, POINT, FACE) |
| `SYNCHRONOUS` | Uses castRayToObjects without snapping |

### Methods

```typescript
async get(config?): Promise<Result | undefined>
```
Performs vertex picking. Dispatches to default or synchronous mode.

```typescript
applySnapping(intersects, snappingClass): void
```
Determines closest snap point for the given class within maxDistance.

```typescript
updatePointer(): void
```
Updates the preview marker position during mouse movement.

```typescript
dispose(): void
```
Cleans up marker and event listeners.
