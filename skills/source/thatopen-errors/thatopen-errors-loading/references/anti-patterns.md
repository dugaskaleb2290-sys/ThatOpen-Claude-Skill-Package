# Loading Errors — Anti-Patterns

Common mistakes that cause IFC loading failures. Each anti-pattern includes the symptom, wrong code, correct code, and the rule to follow.

---

## AP-1: Skipping the Initialization Sequence

**Symptom:** Multiple cascading errors, nothing works.

**Wrong — missing three critical initialization steps:**
```typescript
const ifcLoader = components.get(OBC.IfcLoader);
const model = await ifcLoader.load(data, true, "Building");
```

**Correct — complete initialization sequence:**
```typescript
// 1. FragmentsManager worker
const fragmentsManager = components.get(OBC.FragmentsManager);
fragmentsManager.init(workerURL);

// 2. IfcLoader WASM
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// 3. Render loop
components.init();

// 4. Load and display
const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);
```

**Rule:** ALWAYS follow the initialization sequence: worker init, WASM setup, render loop, then load. Skipping any step causes failures.

---

## AP-2: Hardcoding WASM CDN Version

**Symptom:** Works in development, breaks after `npm update` when web-ifc version changes.

**Wrong:**
```typescript
// Version hardcoded — will mismatch after npm update
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.74/",
    absolute: true,
  },
});
```

**Correct:**
```typescript
// Option A: Let autoSetWasm handle versioning (recommended)
await ifcLoader.setup();

// Option B: Dynamic version from package.json
import { version } from "web-ifc/package.json";
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: `https://unpkg.com/web-ifc@${version}/`,
    absolute: true,
  },
});
```

**Rule:** NEVER hardcode a web-ifc version in WASM URLs. ALWAYS use `autoSetWasm: true` or derive the version dynamically.

---

## AP-3: Using text() Instead of arrayBuffer()

**Symptom:** `Invalid IFC file` or garbled data, even though the file is valid.

**Wrong:**
```typescript
const response = await fetch("/model.ifc");
const text = await response.text();
const encoder = new TextEncoder();
const data = encoder.encode(text);  // Encoding corruption
const model = await ifcLoader.load(data, true, "Building");
```

**Correct:**
```typescript
const response = await fetch("/model.ifc");
const buffer = await response.arrayBuffer();
const data = new Uint8Array(buffer);  // Binary-safe
const model = await ifcLoader.load(data, true, "Building");
```

**Rule:** ALWAYS use `response.arrayBuffer()` for IFC files. NEVER use `response.text()` — text encoding/decoding corrupts binary data.

---

## AP-4: Ignoring setup() Return Promise

**Symptom:** Intermittent failures — sometimes works, sometimes WASM not ready.

**Wrong:**
```typescript
ifcLoader.setup();  // NOT awaited — race condition
const model = await ifcLoader.load(data, true, "Building");
```

**Correct:**
```typescript
await ifcLoader.setup();  // ALWAYS await
const model = await ifcLoader.load(data, true, "Building");
```

**Rule:** ALWAYS `await` the `setup()` call. It is an async operation that initializes WASM. Without awaiting, `load()` may execute before WASM is ready.

---

## AP-5: No Error Handling on Load

**Symptom:** Application crashes on malformed IFC files instead of showing a user-friendly error.

**Wrong:**
```typescript
const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);
```

**Correct:**
```typescript
try {
  const model = await ifcLoader.load(data, true, "Building");
  world.scene.three.add(model.object);
  world.camera.fit(world.scene.three.children);
} catch (error) {
  // Show user-friendly message, log details
  console.error("Failed to load IFC:", error);
  showNotification("The IFC file could not be loaded. It may be corrupt or unsupported.");
}
```

**Rule:** ALWAYS wrap `load()` in a try/catch. IFC files from external sources are unreliable — NEVER let parse errors crash the application.

---

## AP-6: Not Checking Container Dimensions

**Symptom:** Everything initializes correctly, model loads, but viewport is blank.

**Wrong:**
```html
<div id="viewer"></div>
<script>
  const container = document.getElementById("viewer");
  world.renderer = new OBC.SimpleRenderer(components, container);
  // Container has 0px height — renderer canvas is invisible
</script>
```

**Correct:**
```html
<style>
  #viewer { width: 100%; height: 100vh; }
</style>
<div id="viewer"></div>
<script>
  const container = document.getElementById("viewer");
  // Verify dimensions before creating renderer
  if (container.offsetHeight === 0) {
    console.error("Viewer container has zero height");
  }
  world.renderer = new OBC.SimpleRenderer(components, container);
</script>
```

**Rule:** ALWAYS ensure the viewer container has explicit non-zero dimensions via CSS before creating the renderer.

---

## AP-7: Loading Multiple Models Without Coordination

**Symptom:** Models overlap or appear at completely different positions.

**Wrong:**
```typescript
const model1 = await ifcLoader.load(data1, false, "Architecture");
const model2 = await ifcLoader.load(data2, false, "Structure");
// coordinate: false — models not aligned
```

**Correct:**
```typescript
const model1 = await ifcLoader.load(data1, true, "Architecture");
const model2 = await ifcLoader.load(data2, true, "Structure");
// coordinate: true — FragmentsManager aligns via coordination matrices
```

**Rule:** ALWAYS pass `coordinate: true` when loading multiple models that need spatial alignment. The `coordinate` parameter triggers coordination matrix application.

---

## AP-8: Forgetting Disposal

**Symptom:** Memory usage grows with every model load, eventually crashing the browser tab.

**Wrong:**
```typescript
// User loads another file — old model still in memory
async function onFileSelected(file: File) {
  const data = new Uint8Array(await file.arrayBuffer());
  const model = await ifcLoader.load(data, true, file.name);
  world.scene.three.add(model.object);
}
```

**Correct:**
```typescript
let currentModel: OBC.FragmentsModel | null = null;

async function onFileSelected(file: File) {
  // Dispose previous model
  if (currentModel) {
    world.scene.three.remove(currentModel.object);
    currentModel.dispose();
  }

  const data = new Uint8Array(await file.arrayBuffer());
  currentModel = await ifcLoader.load(data, true, file.name);
  world.scene.three.add(currentModel.object);
}

// On application teardown
window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

**Rule:** ALWAYS dispose previous models before loading new ones. ALWAYS call `components.dispose()` on application teardown.

---

## AP-9: Assuming All IFC Types Are Loaded

**Symptom:** "Where are my spaces / openings / MEP elements?"

**Wrong:**
```typescript
const model = await ifcLoader.load(data, true, "Building");
// Expects IFCSPACE to be present — it is excluded by default
const classifier = components.get(OBC.Classifier);
classifier.byCategory();
// No spaces in classification tree
```

**Correct:**
```typescript
ifcLoader.onIfcImporterInitialized.add((importer) => {
  importer.classes.elements.add(WEBIFC.IFCSPACE);
});

const model = await ifcLoader.load(data, true, "Building");
```

**Rule:** ALWAYS review the default element class set when specific element types are missing. The default set is optimized for performance, not completeness.

---

## AP-10: No Timeout on Boolean Operations

**Symptom:** Loading hangs indefinitely on certain IFC files, no error thrown.

**Wrong:**
```typescript
await ifcLoader.setup();
const model = await ifcLoader.load(untrustedData, true, "External");
// May hang forever on complex boolean geometry
```

**Correct:**
```typescript
await ifcLoader.setup();
ifcLoader.settings.webIfc.BOOL_ABORT_THRESHOLD = 5000;
ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;
const model = await ifcLoader.load(untrustedData, true, "External");
```

**Rule:** ALWAYS set `BOOL_ABORT_THRESHOLD` when loading IFC files from untrusted or unknown sources. Without a timeout, a single complex boolean operation can hang the entire application indefinitely.

---

## AP-11: Re-Parsing IFC on Every Page Load

**Symptom:** Slow startup (10-60 seconds) every time the page loads, even for the same model.

**Wrong:**
```typescript
// Every visit parses the full IFC file
const data = new Uint8Array(await fetch("/model.ifc").then(r => r.arrayBuffer()));
const model = await ifcLoader.load(data, true, "Building");
```

**Correct:**
```typescript
// Check cache first
const cached = await loadFromIndexedDB("building.frg");
if (cached) {
  const model = fragmentsManager.load(cached);
  world.scene.three.add(model.object);
} else {
  const data = new Uint8Array(await fetch("/model.ifc").then(r => r.arrayBuffer()));
  const model = await ifcLoader.load(data, true, "Building");
  world.scene.three.add(model.object);
  await saveToIndexedDB("building.frg", model.export());
}
```

**Rule:** ALWAYS convert IFC to Fragments binary once and cache the result. Fragment loading is orders of magnitude faster than IFC parsing. NEVER re-parse IFC files that have not changed.

---

## AP-12: Using Deprecated Packages

**Symptom:** Import errors, missing APIs, TypeScript type conflicts.

**Wrong:**
```typescript
import { IfcViewerAPI } from "web-ifc-viewer";  // DEPRECATED
import { IFCLoader } from "web-ifc-three";       // DEPRECATED
```

**Correct:**
```typescript
import * as OBC from "@thatopen/components";          // current (IfcLoader)
import { IfcImporter } from "@thatopen/fragments";    // current (low-level)
```

**Rule:** NEVER use `web-ifc-viewer` or `web-ifc-three`. These packages are deprecated and incompatible with ThatOpen v3. ALWAYS use `@thatopen/components` or `@thatopen/fragments`.
