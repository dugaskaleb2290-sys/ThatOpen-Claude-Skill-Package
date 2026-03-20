# IfcLoader — Anti-Patterns & Common Failures

## AP-1: Calling load() Before setup()

**Symptom:** WASM initialization error, `Cannot read properties of undefined`, or silent failure returning no model.

**Wrong:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
// MISSING: await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "Building");  // FAILS
```

**Correct:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();  // ALWAYS call setup() first
const model = await ifcLoader.load(data, true, "Building");
```

**Rule:** ALWAYS call `setup()` and await it before calling `load()`. IfcLoader implements `Configurable` — WASM initialization happens in `setup()`, not in the constructor.

---

## AP-2: WASM Version Mismatch

**Symptom:** `RuntimeError: unreachable`, `CompileError`, garbled geometry, or silent parsing failures.

**Wrong:**
```typescript
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.74/",  // WRONG: installed version is 0.0.77
    absolute: true,
  },
});
```

**Correct:**
```typescript
// Option A: Let autoSetWasm handle it
await ifcLoader.setup();  // autoSetWasm: true (default)

// Option B: Match the installed version explicitly
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77/",  // matches package.json
    absolute: true,
  },
});
```

**Rule:** NEVER hardcode a web-ifc version in the WASM path without verifying it matches the installed `web-ifc` npm package version. The WASM binary and the JavaScript API must be from the same version.

---

## AP-3: Missing FragmentsManager Initialization

**Symptom:** `FragmentsManager not initialized`, model loads but does not appear, or worker errors.

**Wrong:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
// MISSING: fragmentsManager.init(workerURL)
const model = await ifcLoader.load(data, true, "Building");
```

**Correct:**
```typescript
const fragmentsManager = components.get(OBC.FragmentsManager);
fragmentsManager.init(workerURL);  // ALWAYS init before loading

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "Building");
```

**Rule:** ALWAYS initialize FragmentsManager with a worker URL before any IFC loading. The fragment system requires a Web Worker for processing.

---

## AP-4: Passing Wrong Data Type to load()

**Symptom:** `TypeError`, empty model, or `Invalid IFC file` error.

**Wrong:**
```typescript
// Passing ArrayBuffer directly
const buffer = await response.arrayBuffer();
const model = await ifcLoader.load(buffer, true, "Building");  // FAILS

// Passing a string
const text = await response.text();
const model = await ifcLoader.load(text, true, "Building");  // FAILS

// Passing a File object
const model = await ifcLoader.load(file, true, "Building");  // FAILS
```

**Correct:**
```typescript
const buffer = await response.arrayBuffer();
const data = new Uint8Array(buffer);  // Convert to Uint8Array
const model = await ifcLoader.load(data, true, "Building");
```

**Rule:** ALWAYS pass IFC data as `Uint8Array`. Convert `ArrayBuffer` with `new Uint8Array(buffer)`. For `File` objects, use `new Uint8Array(await file.arrayBuffer())`.

---

## AP-5: Not Adding Model to Scene

**Symptom:** Model loads successfully (no errors) but nothing appears in the viewer.

**Wrong:**
```typescript
const model = await ifcLoader.load(data, true, "Building");
// MISSING: add to scene
```

**Correct:**
```typescript
const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);  // ALWAYS add model.object to scene
```

**Rule:** ALWAYS add `model.object` to the Three.js scene after loading. The `load()` method creates the model but does not automatically add it to any scene.

---

## AP-6: Forgetting cleanUp() After readIfcFile()

**Symptom:** Memory leak, subsequent loads fail, or stale data from previous model.

**Wrong:**
```typescript
const modelID = await ifcLoader.readIfcFile(data);
const walls = ifcLoader.webIfc.GetLineIDsWithType(modelID, WEBIFC.IFCWALL);
// MISSING: ifcLoader.cleanUp()
```

**Correct:**
```typescript
const modelID = await ifcLoader.readIfcFile(data);
const walls = ifcLoader.webIfc.GetLineIDsWithType(modelID, WEBIFC.IFCWALL);
ifcLoader.cleanUp();  // ALWAYS clean up after raw access
```

**Rule:** ALWAYS call `cleanUp()` after using `readIfcFile()`. The `load()` method calls `cleanUp()` automatically, but `readIfcFile()` does not.

---

## AP-7: Hardcoding WASM Path Without Trailing Slash

**Symptom:** 404 errors loading WASM files, `web-ifc.wasm` not found.

**Wrong:**
```typescript
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77",  // MISSING trailing slash
    absolute: true,
  },
});
```

**Correct:**
```typescript
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77/",  // trailing slash required
    absolute: true,
  },
});
```

**Rule:** ALWAYS include a trailing slash in WASM path strings. The path is used as a directory prefix — `web-ifc.wasm` is appended to it.

---

## AP-8: Re-Parsing IFC on Every Page Load

**Symptom:** Slow startup, unnecessary CPU usage, poor user experience.

**Wrong:**
```typescript
// Every page load re-parses the IFC file
const data = new Uint8Array(await fetch("/model.ifc").then(r => r.arrayBuffer()));
const model = await ifcLoader.load(data, true, "Building");  // slow every time
```

**Correct:**
```typescript
// Convert once, cache as fragments
const cached = await loadFromCache("building.frg");
if (cached) {
  const model = fragmentsManager.load(cached);  // fast reload
  world.scene.three.add(model.object);
} else {
  const data = new Uint8Array(await fetch("/model.ifc").then(r => r.arrayBuffer()));
  const model = await ifcLoader.load(data, true, "Building");
  world.scene.three.add(model.object);
  await saveToCache("building.frg", model.export());
}
```

**Rule:** ALWAYS convert IFC to Fragments binary once and cache the result. Fragment loading is orders of magnitude faster than IFC parsing.

---

## AP-9: Missing components.init() Call

**Symptom:** Model loads and is added to scene, but nothing renders. Viewer shows black or empty.

**Wrong:**
```typescript
const components = new OBC.Components();
// ... world setup, loader setup ...
const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);
// MISSING: components.init()
```

**Correct:**
```typescript
const components = new OBC.Components();
// ... world setup, loader setup ...
components.init();  // starts the render loop
const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);
```

**Rule:** ALWAYS call `components.init()` to start the `requestAnimationFrame` render loop. Without it, the scene never draws.

---

## AP-10: Using Deprecated Packages

**Symptom:** Import errors, missing APIs, incompatible types.

**Wrong:**
```typescript
import { IfcViewerAPI } from "web-ifc-viewer";     // DEPRECATED
import { IFCLoader } from "web-ifc-three";          // DEPRECATED
```

**Correct:**
```typescript
import * as OBC from "@thatopen/components";         // current
import { IfcImporter } from "@thatopen/fragments";   // current (low-level)
```

**Rule:** NEVER use `web-ifc-viewer` or `web-ifc-three` packages. They are deprecated and incompatible with ThatOpen v3. Use `@thatopen/components` (IfcLoader) or `@thatopen/fragments` (IfcImporter).
