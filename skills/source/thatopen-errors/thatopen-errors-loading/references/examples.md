# Loading Errors — Fix Examples

## Example 1: WASM Path Fix

**Problem:** `404 Not Found` on `web-ifc.wasm`.

```typescript
// BEFORE (broken)
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77",  // missing trailing slash
    absolute: true,
  },
});
// Resolves to: https://unpkg.com/web-ifc@0.0.77web-ifc.wasm (WRONG)

// AFTER (fixed)
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77/",  // trailing slash present
    absolute: true,
  },
});
// Resolves to: https://unpkg.com/web-ifc@0.0.77/web-ifc.wasm (CORRECT)
```

---

## Example 2: Version Mismatch Fix

**Problem:** `RuntimeError: unreachable` when loading IFC.

```typescript
// BEFORE (broken) — installed web-ifc is 0.0.77, WASM is 0.0.74
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.74/",  // WRONG version
    absolute: true,
  },
});

// AFTER (fixed) — use autoSetWasm to guarantee version match
await ifcLoader.setup();  // autoSetWasm: true (default) matches npm version

// OR — match version explicitly
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77/",  // matches npm ls web-ifc
    absolute: true,
  },
});
```

---

## Example 3: Complete Initialization Fix

**Problem:** Multiple initialization steps missing, causing cascading failures.

```typescript
// BEFORE (broken) — missing init, setup, and scene add
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

const ifcLoader = components.get(OBC.IfcLoader);
const model = await ifcLoader.load(data, true, "Building");
// Nothing renders, multiple errors in console

// AFTER (fixed) — correct initialization order
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

components.init();  // 1. Start render loop

const fragmentsManager = components.get(OBC.FragmentsManager);
fragmentsManager.init(workerURL);  // 2. Initialize worker

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();  // 3. Initialize WASM

const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);  // 4. Add to scene
world.camera.fit(world.scene.three.children);  // 5. Frame camera
```

---

## Example 4: Worker URL Fix

**Problem:** `Failed to construct 'Worker'` or 404 on worker script.

```typescript
// BEFORE (broken) — wrong worker path
const fragmentsManager = components.get(OBC.FragmentsManager);
fragmentsManager.init("./worker.mjs");  // relative path may not resolve

// AFTER (fixed) — use absolute path or URL constructor
// Option A: absolute path from server root
fragmentsManager.init("/workers/fragment-worker.mjs");

// Option B: construct URL relative to current module
const workerURL = new URL(
  "../node_modules/@thatopen/fragments/dist/worker.mjs",
  import.meta.url
).href;
fragmentsManager.init(workerURL);

// Option C: CDN URL (requires CORS)
fragmentsManager.init(
  "https://unpkg.com/@thatopen/fragments@3.3.6/dist/worker.mjs"
);
```

---

## Example 5: SharedArrayBuffer / COOP-COEP Fix

**Problem:** `SharedArrayBuffer is not defined`, multi-threaded WASM unavailable.

### Vite Configuration
```typescript
// vite.config.ts
export default defineConfig({
  server: {
    headers: {
      "Cross-Origin-Opener-Policy": "same-origin",
      "Cross-Origin-Embedder-Policy": "require-corp",
    },
  },
});
```

### Nginx Configuration
```nginx
server {
    location / {
        add_header Cross-Origin-Opener-Policy same-origin;
        add_header Cross-Origin-Embedder-Policy require-corp;
    }
}
```

### Express Configuration
```typescript
app.use((req, res, next) => {
  res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
  res.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
  next();
});
```

### Fallback — Force Single-Threaded
If you cannot configure headers, force single-threaded mode:
```typescript
// web-ifc direct usage
await ifcApi.Init(undefined, true); // forceSingleThread = true
```

---

## Example 6: Boolean Operation Timeout Fix

**Problem:** Loading hangs on specific IFC files with complex geometry.

```typescript
// BEFORE (broken) — no timeout, hangs indefinitely
await ifcLoader.setup();
const model = await ifcLoader.load(complexIfcData, true, "Complex");
// Never returns...

// AFTER (fixed) — set abort threshold and fast bools
await ifcLoader.setup();
ifcLoader.settings.webIfc.BOOL_ABORT_THRESHOLD = 5000;  // 5 second timeout
ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;         // faster algorithm
const model = await ifcLoader.load(complexIfcData, true, "Complex");
// Elements with impossible booleans are skipped, rest loads fine
```

---

## Example 7: Missing Elements Fix

**Problem:** HVAC elements (ducts, pipes) or spaces not visible after loading.

```typescript
import * as WEBIFC from "web-ifc";

const ifcLoader = components.get(OBC.IfcLoader);

// Add missing categories BEFORE loading
ifcLoader.onIfcImporterInitialized.add((importer) => {
  // MEP elements
  importer.classes.elements.add(WEBIFC.IFCFLOWSEGMENT);
  importer.classes.elements.add(WEBIFC.IFCFLOWFITTING);
  importer.classes.elements.add(WEBIFC.IFCFLOWTERMINAL);
  importer.classes.elements.add(WEBIFC.IFCFLOWCONTROLLER);
  importer.classes.elements.add(WEBIFC.IFCFLOWMOVINGDEVICE);
  importer.classes.elements.add(WEBIFC.IFCFLOWSTORAGEDEVICE);
  importer.classes.elements.add(WEBIFC.IFCFLOWTREATMENTDEVICE);
  importer.classes.elements.add(WEBIFC.IFCPIPESEGMENT);
  importer.classes.elements.add(WEBIFC.IFCDUCTSEGMENT);

  // Spaces (for spatial analysis)
  importer.classes.elements.add(WEBIFC.IFCSPACE);
});

await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "MEP-Model");
```

---

## Example 8: Large Coordinate Fix

**Problem:** Model geometry appears distorted, z-fighting, or camera cannot zoom properly.

```typescript
// BEFORE (broken) — model at UTM coordinates
await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "Building");
// Model at x=500000, y=6000000 — floating-point precision loss

// AFTER (fixed) — translate to origin
await ifcLoader.setup();
ifcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
const model = await ifcLoader.load(data, true, "Building");
// Model centered at origin — GPU-safe coordinates
```

---

## Example 9: Corrupt File Defensive Loading

**Problem:** Unknown IFC files from external sources may be corrupt or malformed.

```typescript
async function safeLoadIfc(
  url: string,
  ifcLoader: OBC.IfcLoader,
  world: OBC.World,
): Promise<OBC.FragmentsModel | null> {
  // 1. Fetch with validation
  const response = await fetch(url);
  if (!response.ok) {
    console.error(`Fetch failed: ${response.status} ${response.statusText}`);
    return null;
  }

  const buffer = await response.arrayBuffer();
  if (buffer.byteLength === 0) {
    console.error("Empty file");
    return null;
  }

  // 2. Quick header check
  const header = new TextDecoder().decode(new Uint8Array(buffer, 0, 20));
  if (!header.startsWith("ISO-10303-21")) {
    console.error("Not a valid STEP IFC file");
    return null;
  }

  // 3. Load with defensive settings
  const data = new Uint8Array(buffer);
  ifcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
  ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;
  ifcLoader.settings.webIfc.BOOL_ABORT_THRESHOLD = 10000;

  try {
    const model = await ifcLoader.load(data, true, "External-Model");
    world.scene.three.add(model.object);
    world.camera.fit(world.scene.three.children);
    return model;
  } catch (error) {
    console.error("IFC loading failed:", error.message);
    return null;
  }
}
```

---

## Example 10: WASM MIME Type Fix for Common Servers

**Problem:** `CompileError: expected magic word` — server returns HTML 404 page instead of WASM binary.

### Express.js
```typescript
import express from "express";
const app = express();

// Serve static files with correct MIME types
app.use(express.static("public", {
  setHeaders: (res, filePath) => {
    if (filePath.endsWith(".wasm")) {
      res.setHeader("Content-Type", "application/wasm");
    }
  },
}));
```

### Webpack (copy WASM to output)
```javascript
// webpack.config.js
const CopyPlugin = require("copy-webpack-plugin");

module.exports = {
  plugins: [
    new CopyPlugin({
      patterns: [
        {
          from: "node_modules/web-ifc/*.wasm",
          to: "wasm/[name][ext]",
        },
      ],
    }),
  ],
};
```

### Vite (WASM in public directory)
```bash
# Copy WASM files to public directory
cp node_modules/web-ifc/web-ifc.wasm public/wasm/
cp node_modules/web-ifc/web-ifc-mt.wasm public/wasm/
```

```typescript
// In code
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "/wasm/", absolute: false },
});
```
