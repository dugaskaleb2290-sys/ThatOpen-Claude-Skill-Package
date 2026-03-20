# Anti-Patterns — Viewer Setup

Version: @thatopen/components 3.3.x, @thatopen/components-front 3.3.x

Common mistakes when setting up a ThatOpen viewer and how to fix them.

---

## AP-001: Missing components.init()

**WRONG:**
```typescript
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);
// Forgot components.init() — nothing renders
```

**CORRECT:**
```typescript
// ... same setup ...
components.init(); // ALWAYS call this to start the render loop
```

**Why:** Without `init()`, the `requestAnimationFrame` loop never starts.
The renderer never draws frames. The camera controls do not update. The
viewport appears blank or frozen.

---

## AP-002: Wrong Setup Order

**WRONG:**
```typescript
// Camera before renderer — camera may not bind to the correct canvas
world.camera = new OBC.OrthoPerspectiveCamera(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.scene = new OBC.SimpleScene(components);
```

**CORRECT:**
```typescript
// ALWAYS: scene first, then renderer, then camera
world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);
```

**Why:** The camera needs the renderer's canvas for mouse/touch event
binding. The renderer needs the scene to exist. ALWAYS follow the order:
scene, renderer, camera.

---

## AP-003: Zero-Size Container

**WRONG:**
```html
<!-- No dimensions — renders as 0x0 -->
<div id="viewer"></div>
```

```typescript
const container = document.getElementById("viewer")!;
world.renderer = new OBC.SimpleRenderer(components, container);
// Canvas is 0x0 — viewport is invisible
```

**CORRECT:**
```html
<div id="viewer" style="width: 100%; height: 100vh;"></div>
```

**Why:** The WebGL canvas inherits its size from the container. A container
with zero width or height produces an invisible or degenerate viewport.
The renderer creates the canvas but cannot render to a zero-size buffer.
ALWAYS ensure the container has explicit dimensions via CSS.

---

## AP-004: Container Not in DOM

**WRONG:**
```typescript
const container = document.createElement("div");
// Container not appended to DOM yet
world.renderer = new OBC.SimpleRenderer(components, container);
// ResizeObserver may not work, canvas not visible
```

**CORRECT:**
```typescript
const container = document.createElement("div");
container.style.width = "100%";
container.style.height = "100vh";
document.body.appendChild(container); // FIRST add to DOM
world.renderer = new OBC.SimpleRenderer(components, container); // THEN create renderer
```

**Why:** The renderer uses `ResizeObserver` to track container dimensions.
If the container is not in the DOM, the observer may not fire, and the
canvas size is indeterminate.

---

## AP-005: Missing scene.setup()

**WRONG:**
```typescript
world.scene = new OBC.SimpleScene(components);
// Forgot setup() — no lights, no background
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);
components.init();
// Scene renders completely black — no lights exist
```

**CORRECT:**
```typescript
world.scene = new OBC.SimpleScene(components);
world.scene.setup(); // ALWAYS call setup() for default lights and background
```

**Why:** `SimpleScene` implements `Configurable`. Without `setup()`, the
scene has no lights and no background. All geometry renders as black
silhouettes against a black background.

---

## AP-006: Not Excluding Grid from PostproductionRenderer

**WRONG:**
```typescript
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.renderer.postproduction.enabled = true;

const grid = components.get(OBC.Grids).create(world);
// Grid is included in post-processing — visual artifacts appear
```

**CORRECT:**
```typescript
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.renderer.postproduction.enabled = true;

const grids = components.get(OBC.Grids);
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);
```

**Why:** The post-processing pipeline applies ambient occlusion and edge
detection to everything in the scene. The infinite grid produces incorrect
depth values that create AO halos and false edges. ALWAYS exclude it.

---

## AP-007: Missing Disposal on Component Unmount

**WRONG (React):**
```tsx
function BIMViewer() {
  useEffect(() => {
    const components = new OBC.Components();
    // ... setup ...
    components.init();
    // No cleanup function — memory leaks on every remount
  }, []);
}
```

**CORRECT (React):**
```tsx
function BIMViewer() {
  useEffect(() => {
    const components = new OBC.Components();
    // ... setup ...
    components.init();

    return () => {
      components.dispose(); // ALWAYS dispose on unmount
    };
  }, []);
}
```

**Why:** Each viewer creates WebGL contexts, GPU buffers, WASM memory,
and event listeners. Without disposal, these accumulate on every React
remount. After a few hot reloads, the browser runs out of WebGL contexts
(limit is typically 8-16 per page) and crashes.

---

## AP-008: Using Components After Disposal

**WRONG:**
```typescript
components.dispose();

// NEVER use references after dispose
const worlds = components.get(OBC.Worlds); // Undefined behavior
world.camera.fit(world.meshes); // Dangling references
```

**CORRECT:**
```typescript
components.dispose();
// Null all references — do NOT use them after dispose
components = null;
world = null;
```

**Why:** After `dispose()`, all internal state is cleaned up. GPU buffers
are freed, event handlers are removed, and WASM memory is released.
Accessing disposed objects leads to null reference errors and potential
use-after-free bugs.

---

## AP-009: Multiple WebGL Contexts

**WRONG:**
```typescript
// Creating and destroying viewers in a loop without proper disposal
for (let i = 0; i < 20; i++) {
  const components = new OBC.Components();
  // ... setup world with renderer ...
  components.init();
  // No dispose — each iteration creates a new WebGL context
}
// Browser limit exceeded — "Too many active WebGL contexts"
```

**CORRECT:**
```typescript
let currentComponents: OBC.Components | null = null;

function createViewer(container: HTMLDivElement) {
  // ALWAYS dispose previous viewer first
  if (currentComponents) {
    currentComponents.dispose();
  }

  currentComponents = new OBC.Components();
  // ... setup ...
  currentComponents.init();
}
```

**Why:** Browsers limit the number of active WebGL contexts (typically
8-16). Each renderer creates one context. Without disposing old contexts,
the limit is quickly reached and new renderers silently fail or force-lose
older contexts.

---

## AP-010: Importing PostproductionRenderer in Node.js

**WRONG:**
```typescript
// In a Node.js script or SSR context
import * as OBCF from "@thatopen/components-front";
// Crashes: requires DOM, WebGL, PointerEvent, ResizeObserver
```

**CORRECT:**
```typescript
// Node.js: use core only
import * as OBC from "@thatopen/components";

// Browser only: use components-front
// Guard with environment check if in SSR/isomorphic code
if (typeof window !== "undefined") {
  const OBCF = await import("@thatopen/components-front");
}
```

**Why:** `@thatopen/components-front` depends on browser APIs
(`HTMLCanvasElement`, `WebGLRenderingContext`, `PointerEvent`,
`ResizeObserver`). Importing it in Node.js causes immediate crashes.

---

## AP-011: Calling init() Before World Is Complete

**WRONG:**
```typescript
const components = new OBC.Components();
components.init(); // Too early — no world exists yet

const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
// The update loop is already running on an incomplete world
```

**CORRECT:**
```typescript
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

components.init(); // ALWAYS last, after all setup is done
```

**Why:** `init()` starts the render loop immediately. If called before
the world is fully set up, the loop tries to update/render incomplete
worlds, potentially throwing errors or producing visual glitches.

---

## AP-012: Hardcoding Container Element ID

**WRONG:**
```typescript
// Fragile — breaks if HTML changes
const container = document.getElementById("viewer")!;
// The ! non-null assertion hides the error if element is missing
```

**CORRECT:**
```typescript
const container = document.getElementById("viewer");
if (!container) {
  throw new Error("Viewer container element #viewer not found in DOM");
}
world.renderer = new OBC.SimpleRenderer(components, container);
```

**Why:** Using the non-null assertion operator (`!`) on
`getElementById` hides the case where the element does not exist.
The renderer constructor then receives `null`, causing a cryptic error
deep in Three.js. ALWAYS validate the container exists before use.
