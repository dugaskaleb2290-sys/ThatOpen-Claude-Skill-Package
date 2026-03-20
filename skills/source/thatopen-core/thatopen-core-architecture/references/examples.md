# Examples — Core Architecture

Version: @thatopen/components 3.3.x

---

## 1. Minimal World Setup (Browser)

```typescript
import * as OBC from "@thatopen/components";

// 1. Create the components container
const components = new OBC.Components();

// 2. Get the Worlds manager via the singleton registry
const worlds = components.get(OBC.Worlds);

// 3. Create a typed world
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();

// 4. Assign scene, renderer, camera
const container = document.getElementById("viewer")!;
world.scene = new OBC.SimpleScene(components);
world.scene.setup(); // Creates default lights and background
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// 5. Start the render loop — ALWAYS required
components.init();

// 6. Optional: add a grid
components.get(OBC.Grids).create(world);
```

## 2. Component Registration Pattern

Every custom component follows this pattern:

```typescript
import * as OBC from "@thatopen/components";

class MyTool extends OBC.Component implements OBC.Disposable {
  static readonly uuid = "a1b2c3d4-e5f6-7890-abcd-ef1234567890" as const;
  enabled = true;
  onDisposed = new OBC.Event<void>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(MyTool.uuid, this);
  }

  dispose(): void {
    this.enabled = false;
    this.onDisposed.trigger();
    this.onDisposed.reset();
  }
}

// Usage — ALWAYS via get(), NEVER via new:
const myTool = components.get(MyTool);
```

## 3. Configurable Component Pattern

Components that need async or complex initialization implement
`Configurable`:

```typescript
import * as OBC from "@thatopen/components";

// IfcLoader is Configurable — ALWAYS call setup() before load()
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup(); // Downloads WASM, initializes web-ifc

// Verify setup completed
console.log(ifcLoader.isSetup); // true

// Now safe to load
const model = await ifcLoader.load(ifcBytes, true, "MyModel");
```

## 4. Updateable Component Pattern

Components that need per-frame updates implement `Updateable`:

```typescript
import * as OBC from "@thatopen/components";

class AnimationController extends OBC.Component implements OBC.Updateable {
  static readonly uuid = "b2c3d4e5-f6a7-8901-bcde-f12345678901" as const;
  enabled = true;
  onBeforeUpdate = new OBC.Event<void>();
  onAfterUpdate = new OBC.Event<void>();

  constructor(components: OBC.Components) {
    super(components);
    components.add(AnimationController.uuid, this);
  }

  // Called automatically by Components every frame when enabled=true
  update(delta?: number): void {
    this.onBeforeUpdate.trigger();
    // Animation logic using delta (seconds since last frame)
    this.onAfterUpdate.trigger();
  }
}
```

## 5. Event System Usage

```typescript
import * as OBC from "@thatopen/components";

const fragments = components.get(OBC.FragmentsManager);

// Subscribe to model loading events
const onLoaded = (model: OBC.FragmentsModel) => {
  console.log("Model loaded:", model.modelId);
};
fragments.onFragmentsLoaded.add(onLoaded);

// Later: unsubscribe to prevent memory leaks
fragments.onFragmentsLoaded.remove(onLoaded);

// Temporarily disable an event (handlers stay registered but do not fire)
fragments.onFragmentsLoaded.enabled = false;

// Re-enable
fragments.onFragmentsLoaded.enabled = true;

// Remove ALL handlers (use on dispose)
fragments.onFragmentsLoaded.reset();
```

## 6. DataMap Reactive Collection

```typescript
import * as OBC from "@thatopen/components";

const worlds = components.get(OBC.Worlds);

// React to new worlds being created
worlds.list.onItemSet.add(({ key, value }) => {
  console.log(`World created: ${key}`);
});

// React to worlds being removed
worlds.list.onItemDeleted.add(({ key }) => {
  console.log(`World removed: ${key}`);
});
```

## 7. Multiple Worlds

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);

// Main 3D viewport
const world3D = worlds.create();
world3D.scene = new OBC.SimpleScene(components);
world3D.scene.setup();
world3D.renderer = new OBC.SimpleRenderer(components, container3D);
world3D.camera = new OBC.OrthoPerspectiveCamera(components);

// Plan view
const worldPlan = worlds.create();
worldPlan.scene = new OBC.SimpleScene(components);
worldPlan.scene.setup();
worldPlan.renderer = new OBC.SimpleRenderer(components, containerPlan);
worldPlan.camera = new OBC.OrthoPerspectiveCamera(components);

// Switch plan camera to orthographic top-down
const planCamera = worldPlan.camera as OBC.OrthoPerspectiveCamera;
planCamera.set("Plan");

components.init(); // Starts render loop for ALL worlds
```

## 8. Proper Disposal

```typescript
import * as OBC from "@thatopen/components";

// Listen for disposal
components.onDisposed.add(() => {
  console.log("All components disposed");
});

// On application shutdown — ALWAYS call this
components.dispose();
// After this call, all components, worlds, renderers, scenes, and
// cameras are cleaned up. The animation loop stops.
// Do NOT use any component references after dispose().
```

## 9. Accessing Three.js Objects

```typescript
import * as OBC from "@thatopen/components";
import * as THREE from "three";

const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// Access underlying Three.js objects via .three
const threeScene: THREE.Scene = world.scene.three;
const threeRenderer: THREE.WebGLRenderer = world.renderer.three;
const threeCamera: THREE.PerspectiveCamera = (world.camera as OBC.OrthoPerspectiveCamera).threePersp;

// Add custom Three.js objects directly
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
threeScene.add(cube);

// Track meshes for raycasting (optional)
world.meshes.add(cube);
```

## 10. Full Application Bootstrap

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

const components = new OBC.Components();

// World with post-processing renderer (browser-only)
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

const container = document.getElementById("viewer")!;
world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// Enable post-processing
world.renderer.postproduction.enabled = true;

// Grid
components.get(OBC.Grids).create(world);

// Fragments manager — ALWAYS init with worker URL
const fragments = components.get(OBC.FragmentsManager);
fragments.init("/workers/fragments-worker.js");

// IFC loader — ALWAYS setup before loading
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// Highlighter — ALWAYS setup with world reference
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Start render loop
components.init();

// Load a model
const response = await fetch("/models/building.ifc");
const data = new Uint8Array(await response.arrayBuffer());
const model = await ifcLoader.load(data, true, "building");

// Frame the model in view
world.camera.fit(world.meshes);

// Cleanup on unmount
window.addEventListener("beforeunload", () => {
  components.dispose();
});
```
