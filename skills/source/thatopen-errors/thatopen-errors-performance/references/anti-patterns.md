# Performance Anti-Patterns

## Anti-Pattern 1: Missing Disposal on Route Change

**WRONG:**
```typescript
// SPA navigation — user leaves the viewer page
function onRouteChange() {
  // Just hide the container... memory still held!
  viewerContainer.style.display = "none";
}
```

**RIGHT:**
```typescript
function onRouteChange() {
  components.dispose();
  viewerContainer.innerHTML = "";
}
```

**Why:** Hiding the container does not free GPU memory. Every BufferGeometry,
Material, Texture, and InstancedMesh remains in VRAM. After several
navigations, the tab crashes from memory exhaustion.

---

## Anti-Pattern 2: Loading Without Disposing Previous Model

**WRONG:**
```typescript
async function handleFileUpload(file: File) {
  const bytes = new Uint8Array(await file.arrayBuffer());
  // Loads a new model every time — old models accumulate!
  const model = await ifcLoader.load(bytes, true, file.name);
}
```

**RIGHT:**
```typescript
let currentModelId: string | null = null;

async function handleFileUpload(file: File) {
  // Dispose previous model first
  if (currentModelId) {
    fragments.disposeModel(currentModelId);
  }

  const bytes = new Uint8Array(await file.arrayBuffer());
  const model = await ifcLoader.load(bytes, true, file.name);
  currentModelId = model.modelId;
}
```

**Why:** Each model load allocates new GPU buffers. Without disposing the
previous model, memory grows linearly with each upload. After 3-4 large
models, the browser tab crashes.

---

## Anti-Pattern 3: Missing Worker Initialization

**WRONG:**
```typescript
const components = new OBC.Components();
const fragments = components.get(OBC.FragmentsManager);
// Forgot fragments.init(workerURL) — operations block main thread or fail
const data = await fragments.getData(items);
```

**RIGHT:**
```typescript
const components = new OBC.Components();
const fragments = components.get(OBC.FragmentsManager);
fragments.init("https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs");
// Now getData runs in worker, off main thread
const data = await fragments.getData(items);
```

**Why:** Without the worker, heavy operations like getData, raycast, and model
loading either run on the main thread (causing UI freezes) or fail silently.
ALWAYS verify `fragments.initialized === true` before operations.

---

## Anti-Pattern 4: Repeatedly Parsing Raw IFC

**WRONG:**
```typescript
// Every page load re-parses the IFC from scratch
async function initViewer() {
  const response = await fetch("/models/building.ifc");
  const bytes = new Uint8Array(await response.arrayBuffer());
  const model = await ifcLoader.load(bytes, true, "Building");
}
```

**RIGHT:**
```typescript
// First time: convert and store as fragments
async function convertAndStore(ifcBytes: Uint8Array) {
  const model = await ifcLoader.load(ifcBytes, true, "Building");
  // Store the fragment binary (IndexedDB, server, etc.)
  await storeBinary(model);
}

// Subsequent loads: load pre-converted fragments (10-100x faster)
async function loadStored() {
  const fragBytes = await retrieveBinary("Building");
  // Load fragment binary directly — skips WASM parsing entirely
}
```

**Why:** IFC parsing involves WASM initialization, schema interpretation, and
geometry tessellation. This is expensive: a 50 MB IFC file can take 10-30
seconds. Fragment loading is FlatBuffers deserialization — typically under
1 second for the same model.

---

## Anti-Pattern 5: Forgetting Three.js Object Disposal

**WRONG:**
```typescript
// Create helpers and overlays
const box = new THREE.BoxHelper(mesh, 0xffff00);
scene.add(box);

// Later, "remove" it by just taking it off screen
scene.remove(box);
// GPU memory for geometry and material still allocated!
```

**RIGHT:**
```typescript
scene.remove(box);
box.geometry.dispose();
box.material.dispose();
```

**Why:** `scene.remove()` only detaches the object from the scene graph. The
GPU buffers remain allocated until `.dispose()` is called. This is a Three.js
fundamental — the browser GC does NOT free GPU resources.

---

## Anti-Pattern 6: Missing Polygon Offset on Overlays

**WRONG:**
```typescript
// Overlay material without polygon offset
const sectionFill = new THREE.MeshBasicMaterial({
  color: 0x0000ff,
  transparent: true,
  opacity: 0.3,
  side: THREE.DoubleSide,
});
// Result: z-fighting flickering where fill overlaps model geometry
```

**RIGHT:**
```typescript
const sectionFill = new THREE.MeshBasicMaterial({
  color: 0x0000ff,
  transparent: true,
  opacity: 0.3,
  side: THREE.DoubleSide,
  polygonOffset: true,
  polygonOffsetFactor: 1,
  polygonOffsetUnits: 1,
});
```

**Why:** Two surfaces at the same depth cause z-fighting — the depth buffer
cannot resolve which is in front, resulting in flickering pixels. Polygon
offset biases the depth of one surface to resolve the ambiguity.

---

## Anti-Pattern 7: Synchronous Bulk Operations

**WRONG:**
```typescript
// Process every element one at a time, blocking main thread
for (const elementId of allElementIds) {
  const data = await fragments.getData({
    [modelId]: new Set([elementId])
  });
  processElement(data);
}
```

**RIGHT:**
```typescript
// Batch all elements into one worker call
const allItems: ModelIdMap = {
  [modelId]: new Set(allElementIds)
};
const allData = await fragments.getData(allItems);
// Process results after single round-trip
for (const [id, items] of Object.entries(allData)) {
  items.forEach(processElement);
}
```

**Why:** Each `getData` call is a round-trip to the worker. For 1000 elements,
that is 1000 message-passing operations instead of 1. Batch into a single
ModelIdMap for one worker call.

---

## Anti-Pattern 8: Not Monitoring Memory in Development

**WRONG:**
```typescript
// No memory monitoring — crashes discovered by users in production
```

**RIGHT:**
```typescript
// Development-only memory monitoring
if (process.env.NODE_ENV === "development") {
  setInterval(() => {
    const mem = (performance as any).memory;
    if (mem && mem.usedJSHeapSize > 2 * 1024 * 1024 * 1024) {
      console.warn("Memory usage exceeds 2 GB — check for leaks");
    }

    const info = renderer.info;
    console.debug(
      `GPU: ${info.memory.geometries} geometries, ` +
      `${info.memory.textures} textures, ` +
      `${info.render.triangles} triangles/frame`
    );
  }, 5000);
}
```

**Why:** Memory leaks are silent until the tab crashes. Proactive monitoring
during development catches leaks early, before they reach production.

---

## Anti-Pattern 9: Worker Version Mismatch

**WRONG:**
```typescript
// package.json has @thatopen/fragments@3.3.6
// but worker URL points to different version
fragments.init(
  "https://unpkg.com/@thatopen/fragments@3.2.0/dist/Worker/worker.mjs"
);
```

**RIGHT:**
```typescript
// ALWAYS match worker URL to installed package version
fragments.init(
  "https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs"
);
```

**Why:** The worker and main thread share a binary protocol (FlatBuffers
schema). Version mismatches cause deserialization failures, corrupted data,
or silent wrong results. ALWAYS keep them in sync.

---

## Anti-Pattern 10: Loading All IFC Classes for Large Models

**WRONG:**
```typescript
// Force-load every IFC class including heavy ones
// This loads IFCOPENINGELEMENT, IFCSPACE, and other geometry-heavy classes
// that are excluded by default for good reason
```

**RIGHT:**
```typescript
// Use default settings — they already exclude heavy classes
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// Only override if you specifically need excluded classes
// and have verified the memory impact
```

**Why:** Some IFC classes (IFCOPENINGELEMENT, IFCSPACE) generate significant
geometry that is rarely needed for visualization. The default import settings
are optimized for the common case. Loading everything can double or triple
memory usage.

---

## Summary: Rules to Prevent Performance Issues

| Rule | Category |
|---|---|
| ALWAYS call `components.dispose()` on teardown | Disposal |
| ALWAYS call `disposeModel()` before loading replacement | Disposal |
| ALWAYS dispose custom Three.js objects manually | Disposal |
| ALWAYS initialize worker before fragment operations | Worker |
| ALWAYS match worker URL to installed fragments version | Worker |
| NEVER re-parse IFC on every load — use fragments | Loading |
| NEVER load all IFC classes for large models | Loading |
| NEVER process elements one-at-a-time — batch via ModelIdMap | Operations |
| ALWAYS use polygon offset for overlay materials | Rendering |
| ALWAYS monitor memory during development | Monitoring |
