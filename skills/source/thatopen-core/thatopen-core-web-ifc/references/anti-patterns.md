# web-ifc Anti-Patterns and Common Failures

## 1. WASM Initialization Failures

### Missing or Wrong WASM Path

**WRONG:**
```typescript
const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init(); // Fails silently or throws — WASM files not found
```

**CORRECT:**
```typescript
const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath("/wasm/"); // Directory containing web-ifc.wasm
await ifcApi.Init();
```

The WASM path directory MUST contain the `.wasm` files matching the installed `web-ifc` version. NEVER hardcode a version-specific path — when the package updates, the WASM files change.

### Calling Methods Before Init

**WRONG:**
```typescript
const ifcApi = new WebIFC.IfcAPI();
const modelID = ifcApi.OpenModel(data); // CRASH: WASM not initialized
```

**CORRECT:**
```typescript
const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath("/wasm/");
await ifcApi.Init();                    // ALWAYS await Init() first
const modelID = ifcApi.OpenModel(data);
```

### Setting WASM Path After Init

**WRONG:**
```typescript
await ifcApi.Init();
ifcApi.SetWasmPath("/wasm/"); // Too late — Init already loaded WASM
```

**CORRECT:**
```typescript
ifcApi.SetWasmPath("/wasm/"); // ALWAYS before Init
await ifcApi.Init();
```

---

## 2. Memory Leaks

### Forgetting to Close Models

**WRONG:**
```typescript
async function processIfc(data: Uint8Array) {
  const modelID = ifcApi.OpenModel(data);
  const walls = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
  return walls; // Model stays open — WASM memory leaked
}
```

**CORRECT:**
```typescript
async function processIfc(data: Uint8Array) {
  const modelID = ifcApi.OpenModel(data);
  try {
    const walls = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
    // Process walls...
    return result;
  } finally {
    ifcApi.CloseModel(modelID); // ALWAYS close in finally block
  }
}
```

### Creating Multiple IfcAPI Instances

**WRONG:**
```typescript
// Each instance loads a separate WASM module (~10MB+)
const api1 = new WebIFC.IfcAPI();
const api2 = new WebIFC.IfcAPI();
const api3 = new WebIFC.IfcAPI();
```

**CORRECT:**
```typescript
// One instance handles all models
const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init();
const model1 = ifcApi.OpenModel(data1);
const model2 = ifcApi.OpenModel(data2);
```

### Not Calling Dispose

**WRONG:**
```typescript
// Application shutdown — IfcAPI still holds WASM memory
window.removeEventListener("unload", cleanup);
```

**CORRECT:**
```typescript
function cleanup() {
  for (const modelID of openModels) {
    ifcApi.CloseModel(modelID);
  }
  ifcApi.Dispose(); // Release entire WASM module
}
```

---

## 3. Vector Access Errors

### Using Array Indexing on WASM Vectors

**WRONG:**
```typescript
const walls = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
for (let i = 0; i < walls.length; i++) {  // .length is undefined
  const wall = ifcApi.GetLine(modelID, walls[i]); // [] returns undefined
}
```

**CORRECT:**
```typescript
const walls = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
for (let i = 0; i < walls.size(); i++) {     // .size() for length
  const wall = ifcApi.GetLine(modelID, walls.get(i)); // .get(i) for access
}
```

This applies to ALL `Vector<T>` returns: `GetAllLines`, `GetLineIDsWithType`, `LoadAllGeometry`, and `FlatMesh.geometries`.

### Spreading WASM Vectors

**WRONG:**
```typescript
const ids = [...ifcApi.GetAllLines(modelID)]; // Spread does not work on WASM vectors
```

**CORRECT:**
```typescript
const vec = ifcApi.GetAllLines(modelID);
const ids: number[] = [];
for (let i = 0; i < vec.size(); i++) {
  ids.push(vec.get(i));
}
```

---

## 4. Geometry Extraction Mistakes

### Wrong Vertex Data Layout

**WRONG:**
```typescript
const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
// Treating as 3 floats per vertex — WRONG, it is 6
const positions = new THREE.BufferAttribute(verts, 3);
```

**CORRECT:**
```typescript
const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
// 6 floats per vertex: [x, y, z, nx, ny, nz]
const vertexCount = verts.length / 6;
const positions = new Float32Array(vertexCount * 3);
const normals = new Float32Array(vertexCount * 3);

for (let i = 0; i < verts.length; i += 6) {
  const j = (i / 6) * 3;
  positions[j] = verts[i]; positions[j+1] = verts[i+1]; positions[j+2] = verts[i+2];
  normals[j] = verts[i+3]; normals[j+1] = verts[i+4]; normals[j+2] = verts[i+5];
}
```

### Ignoring the Transform Matrix

**WRONG:**
```typescript
// Placing geometry without its transform — ends up at wrong position
const mesh = new THREE.Mesh(bufferGeometry, material);
scene.add(mesh);
```

**CORRECT:**
```typescript
const mesh = new THREE.Mesh(bufferGeometry, material);
mesh.applyMatrix4(new THREE.Matrix4().fromArray(pg.flatTransformation));
scene.add(mesh);
```

### Ignoring the Coordination Matrix

**WRONG:**
```typescript
// Loading multiple models without coordination — they overlap incorrectly
const group1 = loadModel(data1);
const group2 = loadModel(data2);
```

**CORRECT:**
```typescript
const group1 = loadModel(data1);
group1.applyMatrix4(new THREE.Matrix4().fromArray(ifcApi.GetCoordinationMatrix(modelID1)));
const group2 = loadModel(data2);
group2.applyMatrix4(new THREE.Matrix4().fromArray(ifcApi.GetCoordinationMatrix(modelID2)));
```

---

## 5. Performance Anti-Patterns

### Flattening on Large Models

**WRONG:**
```typescript
// flatten=true recursively resolves ALL references — extremely slow on large models
const allLines = ifcApi.GetAllLines(modelID);
for (let i = 0; i < allLines.size(); i++) {
  const entity = ifcApi.GetLine(modelID, allLines.get(i), true); // DO NOT flatten every entity
}
```

**CORRECT:**
```typescript
// Use flatten only for specific entities you need fully resolved
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
for (let i = 0; i < wallIDs.size(); i++) {
  const wall = ifcApi.GetLine(modelID, wallIDs.get(i)); // no flatten
  // Resolve specific references manually if needed
}
```

### Using LoadAllGeometry Instead of Streaming

**WRONG:**
```typescript
// Loads ALL geometry into memory at once — crashes on large models
const allMeshes = ifcApi.LoadAllGeometry(modelID);
```

**CORRECT:**
```typescript
// Stream geometry one element at a time — constant memory usage
ifcApi.StreamAllMeshes(modelID, (mesh, index, total) => {
  processGeometry(mesh);
});
```

### Not Using COORDINATE_TO_ORIGIN

**WRONG:**
```typescript
const modelID = ifcApi.OpenModel(data);
// Model at real-world coordinates (e.g., x=500000, y=6000000)
// Causes floating-point precision issues in Three.js rendering
```

**CORRECT:**
```typescript
const modelID = ifcApi.OpenModel(data, { COORDINATE_TO_ORIGIN: true });
// Model translated to origin — safe for GPU rendering
```

---

## 6. Schema-Related Mistakes

### Assuming Schema Version

**WRONG:**
```typescript
// Assuming IFC4 — will fail on IFC2X3 files
const storeys = ifcApi.GetLineIDsWithType(modelID, IFCBUILDINGSTOREY);
```

**CORRECT:**
```typescript
const schema = ifcApi.GetModelSchema(modelID);
console.log(`Loading ${schema} model`);
// Some type constants may differ between IFC2X3 and IFC4
// ALWAYS check schema when handling schema-specific types
```

### Not Checking Available Types

**WRONG:**
```typescript
// Assumes the model contains walls — may return empty vector
const walls = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
const firstWall = ifcApi.GetLine(modelID, walls.get(0)); // Crashes if size() == 0
```

**CORRECT:**
```typescript
const walls = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
if (walls.size() === 0) {
  console.log("No walls in this model");
  return;
}
const firstWall = ifcApi.GetLine(modelID, walls.get(0));
```

---

## 7. Property Access Mistakes

### Accessing Property Values Directly

**WRONG:**
```typescript
const wall = ifcApi.GetLine(modelID, expressID);
console.log(wall.Name); // Logs { type: 1, value: "Wall-001" } — not the string
```

**CORRECT:**
```typescript
const wall = ifcApi.GetLine(modelID, expressID);
console.log(wall.Name?.value); // Logs "Wall-001"
```

IFC property values are wrapped objects with `type` and `value` fields. ALWAYS access `.value` to get the actual data. ALWAYS use optional chaining (`?.`) because properties may be null.

### Unresolved References

**WRONG:**
```typescript
const wall = ifcApi.GetLine(modelID, expressID);
console.log(wall.OwnerHistory.OwningUser); // OwnerHistory is { type: 5, value: 42 } — a reference
```

**CORRECT:**
```typescript
const wall = ifcApi.GetLine(modelID, expressID);
// Option 1: Resolve manually
const ownerHistory = ifcApi.GetLine(modelID, wall.OwnerHistory.value);

// Option 2: Use flatten (only for small, targeted queries)
const wallFlat = ifcApi.GetLine(modelID, expressID, true);
console.log(wallFlat.OwnerHistory.OwningUser); // Now fully resolved
```

References have `type: 5` and a `value` containing the expressID of the referenced entity.
