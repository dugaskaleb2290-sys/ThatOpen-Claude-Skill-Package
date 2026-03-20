# API Reference — Clipper, ClipStyler, ClipEdges, Views

## Clipper

**Package**: `@thatopen/components`
**UUID**: `66290bc5-18c4-4cd1-9379-2e17a0617611`
**Implements**: `Component`, `Createable`, `Disposable`, `Configurable`

### Constructor

```typescript
constructor(components: Components)
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enabled` | `boolean` | `false` | Enables/disables the component |
| `visible` | `boolean` | `true` | Shows/hides all plane helpers |
| `size` | `number` | `5` | Geometric size of plane helpers |
| `material` | `THREE.MeshBasicMaterial` | — | Material for plane helpers |
| `orthogonalY` | `boolean` | `false` | Forces planes orthogonal to Y axis |
| `toleranceOrthogonalY` | `number` | `0.7` | Threshold for Y orthogonality |
| `autoScalePlanes` | `boolean` | `true` | Auto-scale based on camera distance |
| `list` | `DataMap<string, SimplePlane>` | — | All created planes |
| `config` | `ClipperConfigManager` | — | Configuration manager |
| `isSetup` | `boolean` | `false` | Whether setup() has been called |
| `Type` | `Constructor<SimplePlane>` | `SimplePlane` | Plane type to instantiate |

### Methods

```typescript
// Interactive creation via raycast
create(world: World): Promise<SimplePlane | null>

// Programmatic creation — returns plane ID
createFromNormalAndCoplanarPoint(
  world: World,
  normal: THREE.Vector3,
  point: THREE.Vector3
): string

// Delete plane (interactive if no ID, specific if ID given)
delete(world: World, planeId?: string): Promise<void>

// Delete all planes, optionally filtered by type
deleteAll(types?: Set<string>): void

// Initialize with configuration
setup(config?: Partial<ClipperConfig>): void

// Clean up all resources
dispose(): void
```

### Events

| Event | Payload | Description |
|-------|---------|-------------|
| `onBeforeCreate` | `void` | Before plane creation |
| `onAfterCreate` | `SimplePlane` | After plane is created |
| `onBeforeDrag` | `SimplePlane` | Before drag starts |
| `onAfterDrag` | `SimplePlane` | After drag ends |
| `onBeforeDelete` | `void` | Before deletion |
| `onAfterDelete` | `SimplePlane` | After plane is deleted |
| `onBeforeCancel` | `void` | Before creation cancelled |
| `onAfterCancel` | `void` | After creation cancelled |
| `onSetup` | `void` | After setup completes |
| `onDisposed` | `string` | Component disposed |
| `onStateChanged` | `string[]` | State property changed |

---

## SimplePlane

**Package**: `@thatopen/components`

### Constructor

```typescript
constructor(
  components: Components,
  world: World,
  origin: THREE.Vector3,
  normal: THREE.Vector3,
  material: THREE.Material,
  size?: number,          // default: 5
  activateControls?: boolean  // default: true
)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `three` | `THREE.Plane` | Underlying Three.js plane |
| `origin` | `THREE.Vector3` | Plane origin point |
| `normal` | `THREE.Vector3` | Plane normal vector |
| `enabled` | `boolean` | Whether plane clips geometry |
| `visible` | `boolean` | Whether helper is visible |
| `autoScale` | `boolean` | Auto-scale with camera distance |
| `size` | `number` | Helper geometric size |
| `type` | `string` | Custom identifier (default `"default"`) |
| `title` | `string` | Display title |
| `meshes` | `THREE.Mesh[]` | Raycasting meshes (readonly) |
| `planeMaterial` | `THREE.Material` | Helper material |
| `helper` | `THREE.Object3D` | Helper object (readonly) |
| `controls` | `TransformControls` | Drag controls (readonly) |
| `components` | `Components` | Components instance |
| `world` | `World` | Associated world |

### Methods

```typescript
setFromNormalAndCoplanarPoint(normal: THREE.Vector3, point: THREE.Vector3): void
update(): void
dispose(): void
```

### Events

| Event | Description |
|-------|-------------|
| `onDraggingStarted` | Drag begins |
| `onDraggingEnded` | Drag ends |
| `onDisposed` | Plane disposed |

---

## ClipStyler

**Package**: `@thatopen/components-front`
**UUID**: `24dfc306-a3c4-410f-8071-babc4afa5e4d`
**Implements**: `Component`, `Disposable`

### Constructor

```typescript
constructor(components: OBC.Components)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `enabled` | `boolean` | Component active state |
| `world` | `OBC.World \| null` | Current world (MUST set before use) |
| `styles` | `DataMap<string, ClipStyle>` | Named style definitions |
| `list` | `DataMap<string, ClipEdges>` | Managed ClipEdges instances (readonly) |
| `visible` | `boolean` | Show/hide all edges |

### Methods

```typescript
// Create from a raw THREE.Plane
create(plane: THREE.Plane, config?: ClipEdgesCreationConfig): ClipEdges

// Create from a View (camera-synced)
createFromView(view: OBC.View, config?: ClipEdgesCreationConfig): ClipEdges

// Create from a Clipper plane ID
createFromClipping(id: string, config?: ClipEdgesCreationConfig): ClipEdges

// Clean up
dispose(): void
```

### Events

| Event | Description |
|-------|-------------|
| `onDisposed` | Component disposed |

---

## ClipEdges

**Package**: `@thatopen/components-front` (managed by ClipStyler)
**Implements**: `Disposable`

### Constructor

```typescript
constructor(components: OBC.Components, plane: THREE.Plane)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `three` | `THREE.Group` | Group holding edge/fill meshes |
| `plane` | `THREE.Plane` | Associated clipping plane (readonly) |
| `items` | `DataMap<string, ClipEdgesItemStyle>` | Named item style groups |
| `world` | `OBC.World` | Associated world |
| `visible` | `boolean` | Show/hide in scene |

### Methods

```typescript
// Recalculate edges (all or specific groups)
update(groups?: string[]): Promise<void>

// Clean up
dispose(): void
```

### Events

| Event | Description |
|-------|-------------|
| `onDisposed` | Instance disposed |

---

## Types

### ClipStyle

```typescript
interface ClipStyle {
  linesMaterial?: LineMaterial;    // Edge line material
  fillsMaterial?: THREE.Material;  // Fill material for cut surfaces
}
```

### ClipEdgesItemStyle

```typescript
interface ClipEdgesItemStyle {
  style: string;                            // Name of style in clipStyler.styles
  data?: OBC.ClassifierIntersectionInput;   // Filter by classification
}
```

### ClipEdgesCreationConfig

```typescript
interface ClipEdgesCreationConfig {
  link?: boolean;                             // Auto-sync with source plane
  id?: string;                                // Custom identifier
  world?: OBC.World;                          // Override default world
  items?: Record<string, ClipEdgesItemStyle>; // Named item groups with styles
}
```

---

## Views

**Package**: `@thatopen/components`

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `list` | `DataMap<string, View>` | All views (readonly) |
| `enabled` | `boolean` | Component active state |
| `world` | `OBC.World \| undefined` | Default world for views |
| `hasOpenViews` | `boolean` | Whether any view is active (getter) |
| `defaultRange` | `number` | Default range (15 units, static) |

### Methods

```typescript
// Create view from normal vector and point
create(
  normal: THREE.Vector3,
  point: THREE.Vector3,
  config?: CreateViewConfig
): View

// Create from THREE.Plane
createFromPlane(
  plane: THREE.Plane,
  config?: CreateViewConfig
): View

// Create views from IFC building storeys
createFromIfcStoreys(
  config?: CreateViewFromIfcStoreysConfig
): Promise<View[]>

// Create elevation views (front, back, left, right)
createElevations(
  config?: CreateElevationViewsConfig
): Promise<View[]>

// Activate a view by ID
open(id: string): void

// Deactivate a view (or any open view if no ID)
close(id?: string): void
```

### Config Types

```typescript
interface CreateViewConfig {
  id?: string;
  world?: OBC.World;
}

interface CreateViewFromIfcStoreysConfig {
  modelIds?: RegExp[];
  storeyNames?: RegExp[];
  offset?: number;         // default: 0.25
  world?: OBC.World;
}

interface CreateElevationViewsConfig {
  combine?: boolean;       // default: false
  modelIds?: RegExp[];
  world?: OBC.World;
  namingCallback?: (modelId: string) => {
    front: string; back: string; left: string; right: string;
  };
}
```

---

## View

**Package**: `@thatopen/components`

### Constructor

```typescript
constructor(
  components: Components,
  config?: { id?: string; normal?: THREE.Vector3; point?: THREE.Vector3 }
)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | `string` | Unique identifier |
| `plane` | `THREE.Plane` | Primary clipping plane |
| `farPlane` | `THREE.Plane` | Back clipping plane |
| `camera` | `OrthoPerspectiveCamera` | Associated camera |
| `open` | `boolean` | Whether view is active |
| `range` | `number` | Distance between front/back planes |
| `distance` | `number` | Plane constant value |
| `world` | `OBC.World \| null` | Associated world |
| `planesEnabled` | `boolean` | Toggle plane clipping |
| `helpersVisible` | `boolean` | Toggle debug helpers |
| `planeHelperColor` | `THREE.Color` | Primary helper color (set only) |
| `farPlaneHelperColor` | `THREE.Color` | Far helper color (set only) |

### Methods

```typescript
update(): void     // Sync camera and far plane
flip(): void       // Reverse view direction
dispose(): void    // Clean up
```

### Events

| Event | Payload | Description |
|-------|---------|-------------|
| `onStateChanged` | `string[]` | State property changed |
| `onUpdated` | `undefined` | After update |
| `onDisposed` | `undefined` | View disposed |
