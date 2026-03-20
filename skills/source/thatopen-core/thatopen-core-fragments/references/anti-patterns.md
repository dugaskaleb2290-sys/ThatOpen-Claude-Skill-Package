# Fragment System Anti-Patterns

## 1. Missing Worker Initialization

**WRONG — calling fragment operations before init:**

```typescript
const fragments = components.get(OBC.FragmentsManager);
// MISSING: fragments.init(workerURL)

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "test");
// Result: silent failure, undefined behavior, or crash
```

**CORRECT:**

```typescript
const fragments = components.get(OBC.FragmentsManager);
fragments.init("https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs");
// NOW safe to use IfcLoader and all fragment operations
```

**Rule:** ALWAYS call `fragments.init(workerURL)` immediately after getting
the FragmentsManager and BEFORE any model loading or querying.

---

## 2. Worker Version Mismatch

**WRONG — hardcoded worker URL that does not match installed version:**

```typescript
// package.json has @thatopen/fragments@3.3.6
// but worker URL points to a different version:
fragments.init("https://unpkg.com/@thatopen/fragments@3.2.0/dist/Worker/worker.mjs");
// Result: deserialization errors, worker message format mismatches
```

**CORRECT:**

```typescript
// ALWAYS match the worker URL to your installed package version
fragments.init("https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs");
```

**Rule:** ALWAYS verify that the worker URL version matches the
`@thatopen/fragments` version in your `package.json` or lock file.

---

## 3. Forgetting to Dispose Models

**WRONG — loading models without disposing them:**

```typescript
async function loadModel(file: File) {
  const data = new Uint8Array(await file.arrayBuffer());
  const model = await ifcLoader.load(data, true, file.name);
  // User loads 5 more files... no disposal
  // GPU memory grows unbounded, browser tab crashes
}
```

**CORRECT:**

```typescript
async function loadModel(file: File) {
  const data = new Uint8Array(await file.arrayBuffer());
  const model = await ifcLoader.load(data, true, file.name);
  return model;
}

function unloadModel(modelId: string) {
  fragments.disposeModel(modelId);
}

// On page unload or component teardown:
window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

**Rule:** ALWAYS dispose models when they are no longer needed. ALWAYS call
`components.dispose()` on page/app teardown.

---

## 4. Skipping Coordinate Alignment in Multi-Model

**WRONG — loading multiple models without alignment:**

```typescript
const modelA = await ifcLoader.load(dataA, true, "Arch");
const modelB = await ifcLoader.load(dataB, true, "Struct");
const modelC = await ifcLoader.load(dataC, true, "MEP");
// Models appear at completely different positions in 3D space
// because each IFC file has its own world origin
```

**CORRECT:**

```typescript
const modelA = await ifcLoader.load(dataA, true, "Arch");
fragments.baseCoordinationModel = modelA.modelId;
fragments.baseCoordinationMatrix = modelA.coordinationMatrix;

const modelB = await ifcLoader.load(dataB, true, "Struct");
fragments.applyBaseCoordinateSystem(modelB, modelB.coordinationMatrix);

const modelC = await ifcLoader.load(dataC, true, "MEP");
fragments.applyBaseCoordinateSystem(modelC, modelC.coordinationMatrix);
```

**Rule:** NEVER skip coordinate alignment when loading multiple models.
ALWAYS set a base coordination model and align all subsequent models to it.

---

## 5. Using expressID Alone as Identifier

**WRONG — assuming expressIDs are globally unique:**

```typescript
// expressID 42 exists in BOTH modelA and modelB
const expressId = 42;
// Which model does this belong to? Ambiguous!
```

**CORRECT — use ModelIdMap:**

```typescript
// ALWAYS pair element IDs with their model ID
const items: ModelIdMap = {
  [modelA.modelId]: new Set([42]),  // expressID 42 in model A
  [modelB.modelId]: new Set([42])   // expressID 42 in model B (different element!)
};
```

**Rule:** NEVER reference elements by expressID alone. ALWAYS use ModelIdMap
to pair element IDs with their parent model UUID.

---

## 6. Reloading IFC Instead of Cached Fragments

**WRONG — parsing IFC from scratch every session:**

```typescript
// User opens the app → fetch IFC → parse with web-ifc → convert to fragments
// Every. Single. Time. Slow and wasteful.
const model = await ifcLoader.load(ifcBytes, true, "building");
```

**CORRECT — convert once, cache the binary:**

```typescript
// First load: convert IFC to fragments and cache
const model = await ifcLoader.load(ifcBytes, true, "building");
// Store the fragment binary (IndexedDB, server, etc.)

// Subsequent loads: load the cached fragment binary directly
// Skips WASM parsing entirely — 10-100x faster
```

**Rule:** ALWAYS convert IFC to fragments once and store the result. NEVER
re-parse the same IFC file on every page load.

---

## 7. Not Listening to Disposal Events

**WRONG — holding references to disposed models:**

```typescript
const model = await ifcLoader.load(data, true, "test");
const modelRef = model; // Stored reference

// Later: model is disposed elsewhere
fragments.disposeModel(model.modelId);

// Bug: modelRef is now a zombie — GPU resources freed, but reference exists
modelRef.fragments.forEach(f => f.mesh); // undefined behavior
```

**CORRECT — listen to disposal events and clean up references:**

```typescript
const model = await ifcLoader.load(data, true, "test");
let activeModel = model;

fragments.onBeforeDispose.add((disposedModel) => {
  if (disposedModel.modelId === activeModel?.modelId) {
    activeModel = null; // Clear reference
  }
});
```

**Rule:** ALWAYS listen to `onBeforeDispose` if you hold references to
models outside of FragmentsManager. NEVER use a model reference after disposal.

---

## 8. Blocking Main Thread with Fragment Operations

**WRONG — expecting synchronous results:**

```typescript
// getData, raycast, getPositions are ALL async (worker-based)
const data = fragments.getData(items); // Returns Promise, not data!
console.log(data); // Logs: Promise {<pending>}
```

**CORRECT:**

```typescript
const data = await fragments.getData(items);
console.log(data); // Actual data
```

**Rule:** ALWAYS await fragment operation results. All data queries and
raycasting run in the worker thread and return Promises.

---

## Summary Table

| Anti-Pattern | Consequence | Rule |
|---|---|---|
| Missing `init(workerURL)` | Silent failures, crashes | ALWAYS init before any operation |
| Worker version mismatch | Deserialization errors | ALWAYS match worker URL to package version |
| No disposal | Memory leaks, tab crashes | ALWAYS dispose unneeded models |
| No coordinate alignment | Models at wrong positions | ALWAYS align multi-model scenarios |
| expressID without model ID | Ambiguous element references | ALWAYS use ModelIdMap |
| Re-parsing IFC every load | Slow startup (WASM overhead) | ALWAYS cache fragment binaries |
| Zombie model references | Undefined behavior | ALWAYS clean refs on dispose |
| Missing await on operations | Promise instead of data | ALWAYS await async fragment methods |
