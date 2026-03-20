# Performance Examples

## Example 1: Proper Full Application Disposal

```typescript
// SPA route change or component unmount
function teardownViewer(components: OBC.Components) {
  // Single call handles everything
  components.dispose();

  // Clear DOM reference
  const container = document.getElementById("viewer-container");
  if (container) {
    container.innerHTML = "";
  }
}
```

## Example 2: Per-Model Load/Unload Cycle

```typescript
const fragments = components.get(OBC.FragmentsManager);
let currentModelId: string | null = null;

async function loadModel(fragBytes: Uint8Array, world: OBC.World) {
  // ALWAYS dispose previous model before loading new one
  if (currentModelId) {
    fragments.disposeModel(currentModelId);
    currentModelId = null;
  }

  // Load the new model
  const model = await loadFragmentBinary(fragBytes, world);
  currentModelId = model.modelId;
}

async function unloadModel() {
  if (currentModelId) {
    fragments.disposeModel(currentModelId);
    currentModelId = null;
  }
}
```

## Example 3: Worker Initialization with Version Matching

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
const fragments = components.get(OBC.FragmentsManager);

// Match worker URL to your installed @thatopen/fragments version
// Check package.json for the exact version
fragments.init(
  "https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs"
);

// Verify initialization
console.log("Worker ready:", fragments.initialized); // true
```

## Example 4: Memory Monitoring During Development

```typescript
function logMemoryUsage(label: string) {
  const mem = (performance as any).memory;
  if (!mem) {
    console.log(`[${label}] performance.memory not available (non-Chrome)`);
    return;
  }

  console.log(`[${label}] Heap: ${(mem.usedJSHeapSize / 1048576).toFixed(1)} MB`);
  console.log(`[${label}] Total: ${(mem.totalJSHeapSize / 1048576).toFixed(1)} MB`);
  console.log(`[${label}] Limit: ${(mem.jsHeapSizeLimit / 1048576).toFixed(1)} MB`);
}

// Usage: track memory across load/unload cycles
logMemoryUsage("Before load");
const model = await loadModel(data, world);
logMemoryUsage("After load");
fragments.disposeModel(model.modelId);
logMemoryUsage("After dispose");
```

## Example 5: Three.js Renderer Info for GPU Monitoring

```typescript
function logGPUStats(renderer: THREE.WebGLRenderer) {
  const info = renderer.info;
  console.log(`Geometries in GPU: ${info.memory.geometries}`);
  console.log(`Textures in GPU: ${info.memory.textures}`);
  console.log(`Draw calls/frame: ${info.render.calls}`);
  console.log(`Triangles/frame: ${info.render.triangles}`);
}

// Call periodically or after load/unload
logGPUStats(world.renderer.three);
```

## Example 6: Disposing Custom Three.js Objects

```typescript
// Custom overlay mesh added to scene
const overlayGeometry = new THREE.PlaneGeometry(10, 10);
const overlayMaterial = new THREE.MeshBasicMaterial({
  color: 0xff0000,
  transparent: true,
  opacity: 0.3,
  polygonOffset: true,       // Prevent z-fighting
  polygonOffsetFactor: -1,
  polygonOffsetUnits: -1,
});
const overlayMesh = new THREE.Mesh(overlayGeometry, overlayMaterial);
world.scene.three.add(overlayMesh);

// Cleanup — YOUR responsibility (components.dispose() won't handle this)
function disposeOverlay() {
  world.scene.three.remove(overlayMesh);
  overlayGeometry.dispose();
  overlayMaterial.dispose();
  // If material had textures:
  // overlayMaterial.map?.dispose();
}
```

## Example 7: IFC Class Filtering for Memory Reduction

```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// The default configuration already excludes some heavy classes
// like IFCOPENINGELEMENT and IFCSPACE for memory efficiency.
// Review ifcLoader.settings to understand what is included by default.

// Load with defaults (already optimized)
const model = await ifcLoader.load(ifcBytes, true, "Building");
```

## Example 8: Multi-Model Federation with Proper Disposal

```typescript
const fragments = components.get(OBC.FragmentsManager);
const loadedModels: string[] = [];

async function loadDiscipline(
  fragBytes: Uint8Array,
  name: string,
  world: OBC.World
): Promise<string> {
  const model = await loadFragmentBinary(fragBytes, world);

  // Set first model as coordination base
  if (loadedModels.length === 0) {
    fragments.baseCoordinationModel = model.modelId;
    fragments.baseCoordinationMatrix = model.coordinationMatrix;
  } else {
    // Align subsequent models to base
    fragments.applyBaseCoordinateSystem(model, model.coordinationMatrix);
  }

  loadedModels.push(model.modelId);
  return model.modelId;
}

function disposeAllModels() {
  // Dispose in reverse order (last loaded first)
  for (let i = loadedModels.length - 1; i >= 0; i--) {
    fragments.disposeModel(loadedModels[i]);
  }
  loadedModels.length = 0;
}
```

## Example 9: Polygon Offset for Custom Highlight Material

```typescript
// Custom highlight material — MUST have polygon offset
const highlightMaterial = new THREE.MeshBasicMaterial({
  color: 0x00ff00,
  transparent: true,
  opacity: 0.4,
  depthTest: true,
  polygonOffset: true,
  polygonOffsetFactor: 1,
  polygonOffsetUnits: 1,
});

// ThatOpen's built-in highlight system handles polygon offset automatically.
// Only apply manually for custom materials YOU create.
```

## Example 10: Memory-Safe Model Swap Pattern

```typescript
class ModelManager {
  private fragments: OBC.FragmentsManager;
  private activeModels = new Map<string, string>(); // name → modelId

  constructor(components: OBC.Components) {
    this.fragments = components.get(OBC.FragmentsManager);
  }

  async swap(name: string, newFragBytes: Uint8Array, world: OBC.World) {
    // Step 1: Dispose old model if exists
    const oldId = this.activeModels.get(name);
    if (oldId) {
      this.fragments.disposeModel(oldId);
      this.activeModels.delete(name);
    }

    // Step 2: Load new model
    const model = await loadFragmentBinary(newFragBytes, world);
    this.activeModels.set(name, model.modelId);

    return model;
  }

  disposeAll() {
    for (const [name, modelId] of this.activeModels) {
      this.fragments.disposeModel(modelId);
    }
    this.activeModels.clear();
  }
}
```
