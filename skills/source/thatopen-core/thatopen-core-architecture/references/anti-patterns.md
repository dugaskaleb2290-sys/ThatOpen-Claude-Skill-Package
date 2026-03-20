# Anti-Patterns — Core Architecture

Version: @thatopen/components 3.3.x

Common mistakes when working with ThatOpen core architecture and what to
do instead.

---

## AP-001: Direct Component Instantiation

**WRONG:**
```typescript
// NEVER do this — bypasses the singleton registry
const worlds = new OBC.Worlds(components);
const ifcLoader = new OBC.IfcLoader(components);
```

**CORRECT:**
```typescript
// ALWAYS use components.get()
const worlds = components.get(OBC.Worlds);
const ifcLoader = components.get(OBC.IfcLoader);
```

**Why:** `components.get()` implements the lazy singleton pattern. Direct
instantiation creates duplicate instances that are not tracked by the
container, leading to lifecycle bugs and memory leaks.

---

## AP-002: Missing components.init()

**WRONG:**
```typescript
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);
// Forgot components.init() — nothing renders, no update loop runs
```

**CORRECT:**
```typescript
// ... same setup as above ...
components.init(); // ALWAYS call this to start the render loop
```

**Why:** Without `init()`, the `requestAnimationFrame` loop never starts.
No `Updateable` components receive `update()` calls, and the renderer
never draws frames.

---

## AP-003: Missing Disposal

**WRONG:**
```typescript
// Component unmount without cleanup
function destroyViewer() {
  container.innerHTML = "";
  // Memory leak: WebGL context, GPU buffers, WASM memory all leaked
}
```

**CORRECT:**
```typescript
function destroyViewer() {
  components.dispose(); // Disposes ALL components, worlds, renderers
}
```

**Why:** BIM models consume hundreds of megabytes of GPU memory and WASM
heap. Failing to dispose causes browser tabs to crash. ALWAYS call
`components.dispose()` when the viewer is no longer needed.

---

## AP-004: Skipping setup() on Configurable Components

**WRONG:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
// Forgot setup() — WASM not initialized
const model = await ifcLoader.load(data, true, "test"); // Fails silently or throws
```

**CORRECT:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup(); // ALWAYS call setup() first
const model = await ifcLoader.load(data, true, "test");
```

**Why:** `Configurable` components defer their initialization to `setup()`.
Without it, internal state (WASM engine, shader compilation, etc.) is not
ready. ALWAYS check `isSetup` if uncertain.

---

## AP-005: Using Deprecated Packages

**WRONG:**
```typescript
// NEVER use these — they are deprecated and abandoned
import { IfcViewerAPI } from "web-ifc-viewer";
import { IFCLoader } from "web-ifc-three";
import * as OBC from "openbim-components";
```

**CORRECT:**
```typescript
// ALWAYS use the @thatopen scoped packages
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
```

**Why:** `web-ifc-viewer`, `web-ifc-three`, and `openbim-components` are
the predecessor packages. They are no longer maintained. The `@thatopen`
scoped packages (v3) are the current, actively developed versions.

---

## AP-006: Importing components-front in Node.js

**WRONG:**
```typescript
// In a Node.js script or server-side code
import * as OBCF from "@thatopen/components-front"; // Crashes — needs DOM + WebGL
```

**CORRECT:**
```typescript
// In Node.js, only use core
import * as OBC from "@thatopen/components";

// components-front is for browser-only code
// Use it ONLY in browser environments
```

**Why:** `@thatopen/components-front` depends on DOM APIs
(`HTMLCanvasElement`, `WebGLRenderingContext`, `PointerEvent`, etc.) that do
not exist in Node.js.

---

## AP-007: Forgetting to Remove Event Handlers

**WRONG:**
```typescript
function setupHandler() {
  const fragments = components.get(OBC.FragmentsManager);
  fragments.onFragmentsLoaded.add((model) => {
    // This handler is never removed — accumulates on repeated calls
    processModel(model);
  });
}
```

**CORRECT:**
```typescript
const handler = (model: OBC.FragmentsModel) => {
  processModel(model);
};

function setupHandler() {
  const fragments = components.get(OBC.FragmentsManager);
  fragments.onFragmentsLoaded.add(handler);
}

function teardownHandler() {
  const fragments = components.get(OBC.FragmentsManager);
  fragments.onFragmentsLoaded.remove(handler);
}
```

**Why:** Anonymous event handlers cannot be removed. Over time, duplicate
handlers accumulate, causing performance degradation and unexpected behavior
(handlers fire multiple times per event).

---

## AP-008: Using v2 Plan/Section APIs

**WRONG:**
```typescript
// These components DO NOT EXIST in v3
const plans = components.get(OBC.Plans);        // Error
const clipEdges = components.get(OBC.ClipEdges); // Error
```

**CORRECT:**
```typescript
// v3 uses ClipStyler + View for plans and sections
import * as OBCF from "@thatopen/components-front";

const clipper = components.get(OBC.Clipper);
const clipStyler = components.get(OBCF.ClipStyler);
// Create clipping plane, then add edge visualization
```

**Why:** The plan/section system was completely redesigned in v3. `Plans`
and `ClipEdges` are replaced by `ClipStyler` with `createFromClipping()`
and `createFromView()`.

---

## AP-009: Not Initializing FragmentsManager Worker

**WRONG:**
```typescript
const fragments = components.get(OBC.FragmentsManager);
// Forgot fragments.init(workerURL)
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "test"); // May fail or run on main thread
```

**CORRECT:**
```typescript
const fragments = components.get(OBC.FragmentsManager);
fragments.init("/workers/fragments-worker.js"); // ALWAYS init with worker URL
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "test");
```

**Why:** `FragmentsManager` offloads heavy geometry operations to a web
worker. Without `init(workerURL)`, operations either fail or block the
main thread, causing the UI to freeze on large models.

---

## AP-010: Using components After dispose()

**WRONG:**
```typescript
components.dispose();
// NEVER use components or any of its managed instances after this
const worlds = components.get(OBC.Worlds); // Undefined behavior
```

**CORRECT:**
```typescript
components.dispose();
// Set reference to null, create new Components if needed
components = null;
```

**Why:** After `dispose()`, all internal state is cleaned up. Accessing
disposed components leads to undefined behavior, null reference errors,
and potential use-after-free bugs in WASM memory.

---

## AP-011: Hardcoding WASM Paths

**WRONG:**
```typescript
await ifcLoader.setup({
  wasm: { path: "/node_modules/web-ifc/0.0.57/", absolute: true }
});
// Hardcoded version — breaks when web-ifc is updated
```

**CORRECT:**
```typescript
// Let setup() auto-detect, or use a version-agnostic path
await ifcLoader.setup();

// Or if customizing, use a path that matches your build output
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "/wasm/", absolute: true }
});
```

**Why:** Hardcoded version strings in WASM paths break silently when
dependencies are updated. Either use auto-detection or copy WASM files
to a stable path during your build process.
