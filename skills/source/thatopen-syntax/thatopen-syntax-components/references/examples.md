# Examples — Component Syntax

Version: @thatopen/components 3.3.x

---

## 1. Custom Component with Disposable

The minimal custom component pattern:

```typescript
import * as OBC from "@thatopen/components";

class SelectionTracker extends OBC.Component implements OBC.Disposable {
  static readonly uuid = "c3d4e5f6-a7b8-9012-cdef-123456789012" as const;
  enabled = true;

  readonly onDisposed = new OBC.Event<void>();
  readonly onSelectionChanged = new OBC.Event<Set<number>>();

  private _selected = new Set<number>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(SelectionTracker.uuid, this);
  }

  select(ids: number[]): void {
    ids.forEach((id) => this._selected.add(id));
    this.onSelectionChanged.trigger(this._selected);
  }

  clearSelection(): void {
    this._selected.clear();
    this.onSelectionChanged.trigger(this._selected);
  }

  dispose(): void {
    this.enabled = false;
    this._selected.clear();
    this.onDisposed.trigger();
    this.onDisposed.reset();
    this.onSelectionChanged.reset();
  }
}

// Usage — ALWAYS via get()
const tracker = components.get(SelectionTracker);
tracker.select([101, 102, 103]);
```

## 2. Custom Component with Updateable

Per-frame animation component:

```typescript
import * as OBC from "@thatopen/components";
import * as THREE from "three";

class ModelRotator extends OBC.Component
  implements OBC.Updateable, OBC.Disposable {

  static readonly uuid = "d4e5f6a7-b8c9-0123-defa-234567890123" as const;
  enabled = true;
  speed = 0.5; // radians per second

  readonly onBeforeUpdate = new OBC.Event<void>();
  readonly onAfterUpdate = new OBC.Event<void>();
  readonly onDisposed = new OBC.Event<void>();

  private _target: THREE.Object3D | null = null;

  constructor(components: OBC.Components) {
    super(components);
    components.add(ModelRotator.uuid, this);
  }

  setTarget(object: THREE.Object3D): void {
    this._target = object;
  }

  // Called automatically by Components every frame when enabled=true
  update(delta?: number): void {
    if (!this._target || !delta) return;
    this.onBeforeUpdate.trigger();
    this._target.rotation.y += this.speed * delta;
    this.onAfterUpdate.trigger();
  }

  dispose(): void {
    this.enabled = false;
    this._target = null;
    this.onDisposed.trigger();
    this.onDisposed.reset();
    this.onBeforeUpdate.reset();
    this.onAfterUpdate.reset();
  }
}

// Usage
const rotator = components.get(ModelRotator);
rotator.setTarget(someObject);
rotator.speed = 1.0;
// update() is called automatically each frame — no manual loop needed
```

## 3. Custom Component with Configurable

Component with deferred initialization:

```typescript
import * as OBC from "@thatopen/components";

interface AnalyzerConfig {
  maxElements: number;
  includeGeometry: boolean;
  workerUrl: string;
}

class ModelAnalyzer extends OBC.Component
  implements OBC.Configurable<AnalyzerConfig, Partial<AnalyzerConfig>>,
             OBC.Disposable {

  static readonly uuid = "e5f6a7b8-c9d0-1234-efab-345678901234" as const;
  enabled = true;
  isSetup = false;

  config: AnalyzerConfig = {
    maxElements: 10000,
    includeGeometry: false,
    workerUrl: "/workers/analyzer.js",
  };

  readonly onSetup = new OBC.Event<ModelAnalyzer>();
  readonly onDisposed = new OBC.Event<void>();

  private _worker: Worker | null = null;

  constructor(components: OBC.Components) {
    super(components);
    components.add(ModelAnalyzer.uuid, this);
  }

  setup(config?: Partial<AnalyzerConfig>): void {
    if (config) {
      this.config = { ...this.config, ...config };
    }
    this._worker = new Worker(this.config.workerUrl);
    this.isSetup = true;
    this.onSetup.trigger(this);
  }

  analyze(): void {
    if (!this.isSetup) {
      throw new Error("Call setup() before analyze()");
    }
    // Use this._worker...
  }

  dispose(): void {
    this.enabled = false;
    this.isSetup = false;
    this._worker?.terminate();
    this._worker = null;
    this.onDisposed.trigger();
    this.onDisposed.reset();
    this.onSetup.reset();
  }
}

// Usage — ALWAYS call setup() before using
const analyzer = components.get(ModelAnalyzer);
analyzer.setup({ maxElements: 50000 });
console.log(analyzer.isSetup); // true
analyzer.analyze();
```

## 4. Event System Patterns

### Basic Subscribe / Unsubscribe

```typescript
import * as OBC from "@thatopen/components";

const fragments = components.get(OBC.FragmentsManager);

// ALWAYS store handler reference for later removal
const onModelLoaded = (model: OBC.FragmentsModel) => {
  console.log("Loaded:", model.modelId);
};

// Subscribe
fragments.onFragmentsLoaded.add(onModelLoaded);

// Later: unsubscribe (MUST use same reference)
fragments.onFragmentsLoaded.remove(onModelLoaded);
```

### Custom Event with Typed Data

```typescript
import * as OBC from "@thatopen/components";

interface ClashResult {
  modelA: string;
  modelB: string;
  intersections: number;
}

// Create a typed event
const onClashDetected = new OBC.Event<ClashResult>();

// Subscribe with full type safety
onClashDetected.add((result) => {
  // result is typed as ClashResult
  console.log(`${result.intersections} clashes found`);
});

// Trigger with typed data
onClashDetected.trigger({
  modelA: "model-1",
  modelB: "model-2",
  intersections: 42,
});

// Cleanup
onClashDetected.reset();
```

### Temporarily Suppressing Events

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Disable — handlers stay registered but do not fire
fragments.onFragmentsLoaded.enabled = false;

// Load models silently (no event fires)
await ifcLoader.load(data, true, "silent-load");

// Re-enable
fragments.onFragmentsLoaded.enabled = true;
```

### One-Shot Event Pattern

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Self-removing handler for a one-time reaction
const onFirstLoad = (model: OBC.FragmentsModel) => {
  console.log("First model loaded:", model.modelId);
  fragments.onFragmentsLoaded.remove(onFirstLoad);
};

fragments.onFragmentsLoaded.add(onFirstLoad);
```

## 5. DataMap Reactive Patterns

### Watching Component Registry

```typescript
import * as OBC from "@thatopen/components";

// components.list is a DataMap<string, Component>
components.list.onItemSet.add(({ key, value }) => {
  console.log(`Component registered: ${key}`);
});
```

### Watching World Lifecycle

```typescript
const worlds = components.get(OBC.Worlds);

// React to new worlds
worlds.list.onItemSet.add(({ key, value }) => {
  console.log(`World created: ${key}`);
  // Perform setup for the new world
});

// React to world removal
worlds.list.onItemDeleted.add(({ key }) => {
  console.log(`World disposed: ${key}`);
  // Clean up world-specific resources
});

// React to all worlds being cleared
worlds.list.onCleared.add(() => {
  console.log("All worlds cleared");
});
```

### Iterating a DataMap

```typescript
const worlds = components.get(OBC.Worlds);

// Standard Map iteration — works exactly like Map
for (const [id, world] of worlds.list) {
  console.log(`World ${id}:`, world);
}

// Check size
console.log(`Total worlds: ${worlds.list.size}`);

// Check existence
if (worlds.list.has("some-uuid")) {
  const world = worlds.list.get("some-uuid");
}
```

## 6. DataSet Reactive Patterns

### Tracking Meshes in a World

```typescript
// world.meshes is a DataSet<THREE.Mesh> (or Set — depends on version)
// Use DataSet events when available

const meshSet = new OBC.DataSet<THREE.Mesh>();

meshSet.onItemAdded.add((mesh) => {
  console.log("Mesh added:", mesh.name);
});

meshSet.onItemDeleted.add((mesh) => {
  console.log("Mesh removed:", mesh.name);
});

// Standard Set operations trigger events
meshSet.add(someMesh);    // fires onItemAdded
meshSet.delete(someMesh); // fires onItemDeleted
meshSet.clear();          // fires onCleared
```

## 7. Runtime Interface Detection

```typescript
import * as OBC from "@thatopen/components";

function inspectComponent(component: OBC.Component): void {
  console.log("Disposable:", component.isDisposeable());
  console.log("Updateable:", component.isUpdateable());
  console.log("Configurable:", component.isConfigurable());
  console.log("Resizeable:", component.isResizeable());
  console.log("Hideable:", component.isHideable());
  console.log("Serializable:", component.isSerializable());
}

// Conditional disposal with type narrowing
function safeDispose(component: OBC.Component): void {
  if (component.isDisposeable()) {
    component.dispose(); // TypeScript knows this is Disposable
  }
}

// Conditional setup with type narrowing
function ensureSetup(component: OBC.Component): void {
  if (component.isConfigurable()) {
    if (!component.isSetup) {
      component.setup();
    }
  }
}
```

## 8. Full Custom Component with Multiple Interfaces

```typescript
import * as OBC from "@thatopen/components";

interface ToolConfig {
  sensitivity: number;
  snapToGrid: boolean;
}

class MeasurementTool extends OBC.Component
  implements OBC.Disposable,
             OBC.Updateable,
             OBC.Configurable<ToolConfig, Partial<ToolConfig>> {

  static readonly uuid = "f6a7b8c9-d0e1-2345-fabc-456789012345" as const;
  enabled = true;
  isSetup = false;

  config: ToolConfig = {
    sensitivity: 1.0,
    snapToGrid: true,
  };

  readonly onDisposed = new OBC.Event<void>();
  readonly onBeforeUpdate = new OBC.Event<void>();
  readonly onAfterUpdate = new OBC.Event<void>();
  readonly onSetup = new OBC.Event<MeasurementTool>();
  readonly onMeasured = new OBC.Event<number>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(MeasurementTool.uuid, this);
  }

  setup(config?: Partial<ToolConfig>): void {
    if (config) {
      this.config = { ...this.config, ...config };
    }
    this.isSetup = true;
    this.onSetup.trigger(this);
  }

  update(delta?: number): void {
    if (!this.isSetup) return;
    this.onBeforeUpdate.trigger();
    // Per-frame measurement update logic
    this.onAfterUpdate.trigger();
  }

  dispose(): void {
    this.enabled = false;
    this.isSetup = false;
    this.onDisposed.trigger();
    this.onDisposed.reset();
    this.onBeforeUpdate.reset();
    this.onAfterUpdate.reset();
    this.onSetup.reset();
    this.onMeasured.reset();
  }
}
```
