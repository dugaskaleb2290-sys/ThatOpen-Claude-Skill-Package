# Examples — Viewer Setup

Version: @thatopen/components 3.3.x, @thatopen/components-front 3.3.x

---

## 1. Minimal Viewer (SimpleRenderer)

The absolute minimum code to get a working 3D viewport:

```typescript
import * as OBC from "@thatopen/components";

const container = document.getElementById("viewer") as HTMLDivElement;

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();

world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

components.get(OBC.Grids).create(world);
components.init();
```

HTML requirement:

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    * { margin: 0; padding: 0; }
    #viewer { width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <div id="viewer"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

---

## 2. PostproductionRenderer Viewer

Enhanced viewer with ambient occlusion and edge detection:

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

const container = document.getElementById("viewer") as HTMLDivElement;

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// Enable post-processing
world.renderer.postproduction.enabled = true;

// Grid — ALWAYS exclude from post-processing
const grids = components.get(OBC.Grids);
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);

components.init();
```

---

## 3. Full Application Setup with IFC Loading

Complete viewer with model loading, highlighting, and cleanup:

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

const container = document.getElementById("viewer") as HTMLDivElement;

// 1. Components container
const components = new OBC.Components();

// 2. World with post-processing
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

// 3. Scene
world.scene = new OBC.SimpleScene(components);
world.scene.setup();

// 4. Renderer
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.renderer.postproduction.enabled = true;

// 5. Camera
world.camera = new OBC.OrthoPerspectiveCamera(components);

// 6. Grid (excluded from post-processing)
const grids = components.get(OBC.Grids);
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);

// 7. Fragments manager with worker
const fragments = components.get(OBC.FragmentsManager);
fragments.init("/workers/fragments-worker.js");

// 8. IFC loader
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// 9. Highlighter for selection
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// 10. Start render loop
components.init();

// 11. Load a model
async function loadModel(url: string, name: string) {
  const response = await fetch(url);
  const data = new Uint8Array(await response.arrayBuffer());
  const model = await ifcLoader.load(data, true, name);
  // Frame model in view
  world.camera.fit(world.meshes);
  return model;
}

// 12. Cleanup
function dispose() {
  components.dispose();
}

window.addEventListener("beforeunload", dispose);
```

---

## 4. Custom Scene Background and Lighting

Override the default scene setup with custom values:

```typescript
import * as OBC from "@thatopen/components";
import * as THREE from "three";

// ... world creation ...

world.scene = new OBC.SimpleScene(components);
world.scene.setup(); // Get default lights first

// Override background
world.scene.three.background = new THREE.Color(0x202932);

// Add custom lighting
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.8);
hemiLight.position.set(0, 20, 0);
world.scene.three.add(hemiLight);

// Stronger directional light with shadows
const dirLight = new THREE.DirectionalLight(0xffffff, 1.2);
dirLight.position.set(5, 10, 7);
dirLight.castShadow = true;
world.scene.three.add(dirLight);
```

---

## 5. Multiple Worlds (3D + Plan View)

Two viewports sharing the same Components instance:

```typescript
import * as OBC from "@thatopen/components";

const container3D = document.getElementById("view-3d") as HTMLDivElement;
const containerPlan = document.getElementById("view-plan") as HTMLDivElement;

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);

// Main 3D viewport
const world3D = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();
world3D.scene = new OBC.SimpleScene(components);
world3D.scene.setup();
world3D.renderer = new OBC.SimpleRenderer(components, container3D);
world3D.camera = new OBC.OrthoPerspectiveCamera(components);

// Plan viewport (orthographic top-down)
const worldPlan = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();
worldPlan.scene = new OBC.SimpleScene(components);
worldPlan.scene.setup();
worldPlan.renderer = new OBC.SimpleRenderer(components, containerPlan);
worldPlan.camera = new OBC.OrthoPerspectiveCamera(components);

// Switch plan camera to top-down orthographic
const planCamera = worldPlan.camera as OBC.OrthoPerspectiveCamera;
planCamera.set("Plan");

// Add grids to both
const grids = components.get(OBC.Grids);
grids.create(world3D);
grids.create(worldPlan);

// Single init() starts both worlds
components.init();
```

---

## 6. React Integration

```tsx
import { useEffect, useRef } from "react";
import * as OBC from "@thatopen/components";

function BIMViewer() {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const components = new OBC.Components();
    const worlds = components.get(OBC.Worlds);
    const world = worlds.create<
      OBC.SimpleScene,
      OBC.OrthoPerspectiveCamera,
      OBC.SimpleRenderer
    >();

    world.scene = new OBC.SimpleScene(components);
    world.scene.setup();
    world.renderer = new OBC.SimpleRenderer(components, container);
    world.camera = new OBC.OrthoPerspectiveCamera(components);

    components.get(OBC.Grids).create(world);
    components.init();

    // Cleanup on unmount — ALWAYS dispose
    return () => {
      components.dispose();
    };
  }, []);

  return <div ref={containerRef} style={{ width: "100%", height: "100vh" }} />;
}
```

---

## 7. Vite Project Setup

### package.json

```json
{
  "name": "thatopen-viewer",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "dependencies": {
    "@thatopen/components": "^3.3.3",
    "@thatopen/components-front": "^3.3.3",
    "@thatopen/fragments": "^3.3.6",
    "three": ">=0.175.0",
    "web-ifc": ">=0.0.74"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "vite": "^5.4.0"
  }
}
```

### vite.config.ts

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    {
      name: "coop-coep-headers",
      configureServer(server) {
        server.middlewares.use((_req, res, next) => {
          res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
          res.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
          next();
        });
      },
    },
  ],
  optimizeDeps: {
    exclude: ["web-ifc"],
  },
  worker: {
    format: "es",
  },
});
```

---

## 8. Accessing the WebGL Context

For advanced rendering needs (custom shaders, reading pixels):

```typescript
const gl = world.renderer.three.getContext();

// Enable screenshot capture
world.renderer = new OBC.SimpleRenderer(components, container, {
  preserveDrawingBuffer: true,
});

// Take screenshot
function takeScreenshot(): string {
  world.renderer.three.render(world.scene.three, world.camera.three);
  return world.renderer.three.domElement.toDataURL("image/png");
}
```

---

## 9. Renderer Event Hooks

Hook into the render loop for custom rendering:

```typescript
// Run code before each frame render
world.renderer.onBeforeUpdate.add(() => {
  // Update custom animations, helpers, etc.
});

// Run code after each frame render
world.renderer.onAfterUpdate.add(() => {
  // Post-render overlay, HUD, etc.
});

// React to viewport resize
world.renderer.onResize.add((newSize: THREE.Vector2) => {
  console.log(`Viewport resized to: ${newSize.x}x${newSize.y}`);
});
```
