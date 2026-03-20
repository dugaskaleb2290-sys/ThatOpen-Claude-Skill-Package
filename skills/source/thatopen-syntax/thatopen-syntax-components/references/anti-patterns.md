# Anti-Patterns — Component Syntax

Version: @thatopen/components 3.3.x

Common mistakes when using ThatOpen components, events, and lifecycle
interfaces — and what to do instead.

---

## AP-001: Direct Component Instantiation

**WRONG:**
```typescript
// NEVER do this — bypasses the singleton registry
const worlds = new OBC.Worlds(components);
const ifcLoader = new OBC.IfcLoader(components);
const myTool = new MyTool(components);
```

**CORRECT:**
```typescript
// ALWAYS use components.get()
const worlds = components.get(OBC.Worlds);
const ifcLoader = components.get(OBC.IfcLoader);
const myTool = components.get(MyTool);
```

**Why:** `components.get()` implements lazy singleton creation. Direct
instantiation creates duplicate instances not tracked by the container,
leading to lifecycle bugs, duplicate state, and memory leaks.

---

## AP-002: Missing Disposal in Custom Components

**WRONG:**
```typescript
class BadComponent extends OBC.Component implements OBC.Disposable {
  static readonly uuid = "..." as const;
  enabled = true;
  onDisposed = new OBC.Event<void>();
  onCustomEvent = new OBC.Event<string>();
  private _data = new Map<string, object>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(BadComponent.uuid, this);
  }

  dispose(): void {
    // Missing: does not set enabled=false
    // Missing: does not clear _data
    // Missing: does not reset onCustomEvent
    this.onDisposed.trigger();
    this.onDisposed.reset();
  }
}
```

**CORRECT:**
```typescript
class GoodComponent extends OBC.Component implements OBC.Disposable {
  static readonly uuid = "..." as const;
  enabled = true;
  readonly onDisposed = new OBC.Event<void>();
  readonly onCustomEvent = new OBC.Event<string>();
  private _data = new Map<string, object>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(GoodComponent.uuid, this);
  }

  dispose(): void {
    this.enabled = false;           // Prevent further use
    this._data.clear();             // Release data references
    this.onDisposed.trigger();      // Notify listeners
    this.onDisposed.reset();        // Clear all handlers
    this.onCustomEvent.reset();     // Clear ALL owned events
  }
}
```

**Why:** Failing to reset all events causes handler accumulation. Failing
to clear internal data structures causes memory leaks. Failing to set
`enabled = false` allows the component to continue processing after disposal.

---

## AP-003: Anonymous Event Handlers (Cannot Be Removed)

**WRONG:**
```typescript
function setupHandler() {
  const fragments = components.get(OBC.FragmentsManager);

  // Anonymous arrow function — cannot be removed later
  fragments.onFragmentsLoaded.add((model) => {
    processModel(model);
  });
}

// Called multiple times → duplicate handlers accumulate
setupHandler();
setupHandler();
setupHandler();
// processModel now fires 3 times per event!
```

**CORRECT:**
```typescript
// Store handler reference at a stable scope
const onModelLoaded = (model: OBC.FragmentsModel) => {
  processModel(model);
};

function setupHandler() {
  const fragments = components.get(OBC.FragmentsManager);
  fragments.onFragmentsLoaded.add(onModelLoaded);
}

function teardownHandler() {
  const fragments = components.get(OBC.FragmentsManager);
  fragments.onFragmentsLoaded.remove(onModelLoaded);
}
```

**Why:** `Event.remove()` compares function references. Anonymous functions
create a new reference each time, so they can never be matched for removal.
This causes handler accumulation, performance degradation, and duplicate
side effects.

---

## AP-004: Forgetting to Reset Events on Dispose

**WRONG:**
```typescript
dispose(): void {
  this.enabled = false;
  this.onDisposed.trigger();
  this.onDisposed.reset();
  // Forgot to reset onDataChanged, onError, onProgress...
}
```

**CORRECT:**
```typescript
dispose(): void {
  this.enabled = false;
  this.onDisposed.trigger();
  // Reset EVERY event this component owns
  this.onDisposed.reset();
  this.onDataChanged.reset();
  this.onError.reset();
  this.onProgress.reset();
}
```

**Why:** Unreset events retain references to handler functions, which in
turn retain references to their closures. This prevents garbage collection
of potentially large object graphs. ALWAYS reset every event in dispose().

---

## AP-005: Missing components.add() in Constructor

**WRONG:**
```typescript
class BrokenComponent extends OBC.Component {
  static readonly uuid = "..." as const;
  enabled = true;

  constructor(components: OBC.Components) {
    super(components);
    // Missing: components.add(BrokenComponent.uuid, this)
  }
}

// components.get(BrokenComponent) creates a new instance every time
// because it is never registered in the list
```

**CORRECT:**
```typescript
class WorkingComponent extends OBC.Component {
  static readonly uuid = "..." as const;
  enabled = true;

  constructor(components: OBC.Components) {
    super(components);
    components.add(WorkingComponent.uuid, this);
  }
}
```

**Why:** Without `components.add()`, the component is never registered in
`components.list`. The `get()` method will create a new instance on every
call, breaking the singleton pattern and causing duplicate instances.

---

## AP-006: Missing Static UUID

**WRONG:**
```typescript
class NoUuidComponent extends OBC.Component {
  // Missing static uuid!
  enabled = true;

  constructor(components: OBC.Components) {
    super(components);
    // components.add(???, this) — no uuid to register with
  }
}
```

**CORRECT:**
```typescript
class ProperComponent extends OBC.Component {
  static readonly uuid = "a1b2c3d4-e5f6-7890-abcd-ef1234567890" as const;
  enabled = true;

  constructor(components: OBC.Components) {
    super(components);
    components.add(ProperComponent.uuid, this);
  }
}
```

**Why:** The static `uuid` is the registry key. Without it, the component
cannot be registered or retrieved. ALWAYS use a real UUID (not a random
string) and add `as const` for type narrowing.

---

## AP-007: Using Configurable Component Before setup()

**WRONG:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
// Forgot to call setup() — WASM not initialized
const model = await ifcLoader.load(data, true, "test"); // Fails
```

**CORRECT:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup(); // ALWAYS call setup() first
const model = await ifcLoader.load(data, true, "test");
```

**ALWAYS check `isSetup` if unsure:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
if (!ifcLoader.isSetup) {
  await ifcLoader.setup();
}
```

**Why:** `Configurable` components defer their initialization to `setup()`.
Internal state (WASM engine, shaders, workers) is not ready until setup
completes. Calling methods before setup leads to silent failures or errors.

---

## AP-008: Not Implementing Disposable for Resource-Holding Components

**WRONG:**
```typescript
class LeakyComponent extends OBC.Component {
  static readonly uuid = "..." as const;
  enabled = true;

  private _canvas: HTMLCanvasElement;
  private _worker: Worker;
  private _handlers = new Map<string, Function>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(LeakyComponent.uuid, this);
    this._canvas = document.createElement("canvas");
    this._worker = new Worker("/worker.js");
  }
  // No dispose() — canvas, worker, and handlers are NEVER cleaned up
}
```

**CORRECT:**
```typescript
class CleanComponent extends OBC.Component implements OBC.Disposable {
  static readonly uuid = "..." as const;
  enabled = true;
  readonly onDisposed = new OBC.Event<void>();

  private _canvas: HTMLCanvasElement;
  private _worker: Worker;

  constructor(components: OBC.Components) {
    super(components);
    components.add(CleanComponent.uuid, this);
    this._canvas = document.createElement("canvas");
    this._worker = new Worker("/worker.js");
  }

  dispose(): void {
    this.enabled = false;
    this._canvas.remove();
    this._worker.terminate();
    this.onDisposed.trigger();
    this.onDisposed.reset();
  }
}
```

**Why:** ALWAYS implement `Disposable` if your component creates DOM elements,
Web Workers, WebGL resources, event subscriptions, or any other resources
that need explicit cleanup. Without disposal, `components.dispose()` cannot
clean up your component, causing memory leaks.

---

## AP-009: Subscribing to DataMap/DataSet Without Cleanup

**WRONG:**
```typescript
function watchWorlds() {
  const worlds = components.get(OBC.Worlds);

  // Anonymous handlers on reactive collections — never cleaned up
  worlds.list.onItemSet.add(({ key }) => {
    console.log("World added:", key);
  });
}
```

**CORRECT:**
```typescript
const onWorldAdded = ({ key, value }: { key: string; value: any }) => {
  console.log("World added:", key);
};

function watchWorlds() {
  const worlds = components.get(OBC.Worlds);
  worlds.list.onItemSet.add(onWorldAdded);
}

function unwatchWorlds() {
  const worlds = components.get(OBC.Worlds);
  worlds.list.onItemSet.remove(onWorldAdded);
}
```

**Why:** DataMap and DataSet events follow the same rules as regular events.
Anonymous handlers cannot be removed. ALWAYS store references and clean up
when no longer needed.

---

## AP-010: Using Components After dispose()

**WRONG:**
```typescript
components.dispose();
// NEVER use components or managed instances after this
const worlds = components.get(OBC.Worlds); // Undefined behavior
```

**CORRECT:**
```typescript
components.dispose();
// Null out the reference — create new Components if needed
let componentsRef: OBC.Components | null = components;
componentsRef.dispose();
componentsRef = null;
```

**Why:** After `dispose()`, all internal state is cleaned up. Accessing
disposed components leads to undefined behavior, null reference errors,
and potential use-after-free bugs in WASM memory.
