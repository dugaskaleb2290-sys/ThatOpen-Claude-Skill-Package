# Viewer Builder — Common Scaffolding Mistakes

## AP-001: Missing COOP/COEP Headers

**Symptom**: WASM operations fail silently, `SharedArrayBuffer is not
defined` errors in console, or IFC loading hangs indefinitely.

**Cause**: The Vite dev server does not set cross-origin isolation headers
by default. Without them, the browser disables `SharedArrayBuffer`.

**Fix**: ALWAYS include the COOP/COEP plugin in `vite.config.ts`:

```typescript
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
```

For production, configure these headers on your web server.

---

## AP-002: web-ifc Not Excluded from optimizeDeps

**Symptom**: `Cannot find module 'web-ifc'` or WASM file 404 errors
during development. The WASM module fails to load after Vite pre-bundles it.

**Cause**: Vite's dependency pre-bundling transforms the web-ifc module
in a way that breaks its WASM loader.

**Fix**: ALWAYS exclude web-ifc:

```typescript
optimizeDeps: {
  exclude: ["web-ifc"],
},
```

---

## AP-003: Missing BUI.Manager.init()

**Symptom**: `bim-toolbar`, `bim-panel`, `bim-grid` and other custom
elements render as empty boxes with no styling or behavior.

**Cause**: ThatOpen UI components are Lit-based web components that must
be registered before use.

**Fix**: ALWAYS call `BUI.Manager.init()` before creating any `bim-*`
element:

```typescript
import * as BUI from "@thatopen/ui";
BUI.Manager.init(); // MUST be first
```

---

## AP-004: Wrong Setup Order (Camera Before Renderer)

**Symptom**: Camera controls do not work, black viewport, or TypeScript
errors about undefined properties.

**Cause**: The camera needs the renderer's DOM element for event binding.
Assigning camera before renderer leaves it unbound.

**Fix**: ALWAYS follow: scene -> renderer -> camera:

```typescript
// CORRECT
world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// WRONG — camera before renderer
world.camera = new OBC.OrthoPerspectiveCamera(components);
world.renderer = new OBCF.PostproductionRenderer(components, container);
```

---

## AP-005: Missing scene.setup() Call

**Symptom**: The viewport renders but the scene is completely dark. No
lights, no visible geometry even after loading a model.

**Cause**: `SimpleScene` implements `Configurable`. Without `setup()`,
no default directional or ambient lights are created.

**Fix**: ALWAYS call `setup()` immediately after assigning the scene:

```typescript
world.scene = new OBC.SimpleScene(components);
world.scene.setup(); // Creates default lights
```

---

## AP-006: Grid Not Excluded from PostproductionRenderer

**Symptom**: The grid renders with dark AO halos, thick outlines, and
visual artifacts. It looks broken.

**Cause**: PostproductionRenderer applies ambient occlusion and edge
detection to ALL objects in the scene, including the grid.

**Fix**: ALWAYS exclude the grid:

```typescript
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);
```

---

## AP-007: Calling components.init() Too Early

**Symptom**: Render loop starts but nothing is visible. Console may show
errors about null references in the update loop.

**Cause**: `components.init()` starts the `requestAnimationFrame` loop.
If scene, renderer, or camera are not yet assigned, the loop tries to
render an incomplete world.

**Fix**: ALWAYS call `init()` AFTER all world components are configured:

```typescript
// All setup done
world.scene = ...
world.renderer = ...
world.camera = ...
// Grid, IfcLoader, Highlighter configured

components.init(); // LAST setup call
```

---

## AP-008: Passing File Object to ifcLoader.load()

**Symptom**: TypeError or silent failure when loading an IFC file from
a file input.

**Cause**: `ifcLoader.load()` requires `Uint8Array`, not `File`,
`Blob`, or `ArrayBuffer`.

**Fix**: ALWAYS convert to `Uint8Array`:

```typescript
// WRONG
const model = await ifcLoader.load(file, true, "model");

// WRONG
const buffer = await file.arrayBuffer();
const model = await ifcLoader.load(buffer, true, "model");

// CORRECT
const buffer = await file.arrayBuffer();
const data = new Uint8Array(buffer);
const model = await ifcLoader.load(data, true, "model");
```

---

## AP-009: Skipping ifcLoader.setup()

**Symptom**: `ifcLoader.load()` throws an error about WASM not being
initialized, or returns undefined.

**Cause**: IfcLoader implements `Configurable`. The `setup()` method
initializes the web-ifc WASM engine. Without it, there is no IFC parser.

**Fix**: ALWAYS await `setup()` before any `load()` call:

```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup(); // REQUIRED — initializes WASM
// Now load() is safe to call
```

---

## AP-010: Missing Peer Dependencies

**Symptom**: Import errors at runtime, missing module warnings during
build, or type errors in the editor.

**Cause**: `@thatopen/components` lists `three`, `web-ifc`, and
`@thatopen/fragments` as peer dependencies. npm does not always install
peers automatically.

**Fix**: ALWAYS install ALL packages explicitly:

```bash
npm install @thatopen/components@^3.3.3 \
  @thatopen/components-front@^3.3.3 \
  @thatopen/fragments@^3.3.6 \
  @thatopen/ui@^3.3.3 \
  @thatopen/ui-obc@^3.3.3 \
  three@^0.175.0 \
  web-ifc@^0.0.77
```

NEVER rely on transitive peer dependency installation.

---

## AP-011: Zero-Size Container

**Symptom**: The WebGL canvas exists in the DOM but is invisible. No
3D content renders. No errors in console.

**Cause**: The container element has zero width or height. The renderer
creates a canvas matching the container dimensions.

**Fix**: ALWAYS ensure the container has explicit non-zero dimensions:

```css
/* For bim-viewport in a bim-grid layout, the grid handles sizing */
/* For a raw div, set explicit dimensions: */
#viewer { width: 100vw; height: 100vh; }
```

NEVER use `display: none` on the container at creation time.

---

## AP-012: Missing Disposal

**Symptom**: Memory usage grows on page navigation. Browser tab crashes
after loading several models. GPU memory is not released.

**Cause**: BIM models consume hundreds of MB of GPU memory. Without
disposal, WebGL contexts, textures, and geometry buffers leak.

**Fix**: ALWAYS dispose on teardown:

```typescript
window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

For SPA frameworks, use lifecycle hooks:
- React: `useEffect` cleanup function
- Vue: `onUnmounted`
- Angular: `ngOnDestroy`

---

## AP-013: Using worker.format Other Than "es"

**Symptom**: Fragment workers fail to load. Console shows syntax errors
or module loading failures in worker context.

**Cause**: Fragment workers use ES module imports. Without
`worker: { format: "es" }`, Vite may bundle workers as IIFE or CJS.

**Fix**: ALWAYS set worker format in `vite.config.ts`:

```typescript
worker: {
  format: "es",
},
```

---

## AP-014: Hardcoding WASM Version in CDN Path

**Symptom**: IFC loading fails after updating `web-ifc` package. WASM
file returns 404 from CDN. Or WASM version mismatch causes parse errors.

**Cause**: The CDN URL contains a hardcoded version that no longer
matches the installed `web-ifc` npm package.

**Fix**: Either use `autoSetWasm: true` (default) or derive the version
dynamically:

```typescript
// BEST: use automatic resolution
await ifcLoader.setup(); // autoSetWasm: true

// OR: if manual path is needed, don't hardcode
import { version } from "web-ifc/package.json";
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: `https://unpkg.com/web-ifc@${version}/`, absolute: true },
});
```
