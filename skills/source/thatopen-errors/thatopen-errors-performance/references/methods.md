# Performance Methods Reference

## Disposal Methods

### components.dispose()

Full application disposal. Disposes all registered components in dependency
order, with FragmentsManager disposed last.

```typescript
components.dispose(): void
```

- Sets `components.enabled = false` (stops render loop)
- Iterates `components.list` and calls `dispose()` on each Disposable
- Disposes FragmentsManager last
- Fires `components.onDisposed` event
- After calling, the Components instance is no longer usable

### FragmentsManager.disposeModel(modelId)

Dispose a single model while keeping the viewer active.

```typescript
fragments.disposeModel(modelId: string): void
```

- Fires `onBeforeDispose` with the FragmentsModel
- Removes all Fragment meshes from the scene
- Disposes GPU geometry buffers for each fragment
- Cleans up worker state for the model
- Removes model from `fragments.list`

### Three.js Disposal Methods

| Object | Method | Notes |
|---|---|---|
| `BufferGeometry` | `geometry.dispose()` | Frees GPU vertex/index buffers |
| `Material` | `material.dispose()` | Frees compiled shader program |
| `Texture` | `texture.dispose()` | Frees GPU texture memory |
| `WebGLRenderTarget` | `target.dispose()` | Frees framebuffer |
| `WebGLRenderer` | `renderer.dispose()` | Frees WebGL context |
| `InstancedMesh` | `mesh.dispose()` | Disposes geometry + material |

**Material texture maps that need disposal:**
- `material.map` (diffuse)
- `material.normalMap`
- `material.aoMap`
- `material.emissiveMap`
- `material.metalnessMap`
- `material.roughnessMap`
- `material.envMap`
- `material.lightMap`
- `material.bumpMap`
- `material.displacementMap`
- `material.alphaMap`

### Scene Removal

ALWAYS remove objects from the scene graph before disposing:

```typescript
scene.remove(object);       // Remove from parent
object.geometry.dispose();  // Then dispose GPU resources
object.material.dispose();
```

## Memory Monitoring APIs

### performance.memory (Chrome only)

```typescript
interface MemoryInfo {
  jsHeapSizeLimit: number;    // Maximum heap size (bytes)
  totalJSHeapSize: number;    // Total allocated heap (bytes)
  usedJSHeapSize: number;     // Currently used heap (bytes)
}

// Usage
const mem = (performance as any).memory;
console.log(`Used: ${(mem.usedJSHeapSize / 1048576).toFixed(1)} MB`);
console.log(`Total: ${(mem.totalJSHeapSize / 1048576).toFixed(1)} MB`);
console.log(`Limit: ${(mem.jsHeapSizeLimit / 1048576).toFixed(1)} MB`);
```

**Note:** Only available in Chrome/Chromium. Returns `undefined` in Firefox
and Safari. Use for development profiling, not production monitoring.

### performance.measureUserAgentSpecificMemory() (Cross-browser)

```typescript
// Requires cross-origin isolation headers
const result = await performance.measureUserAgentSpecificMemory();
console.log(`Total: ${(result.bytes / 1048576).toFixed(1)} MB`);
```

### WebGL Memory Estimation

```typescript
// Get GPU memory info (WEBGL_debug_renderer_info extension)
const gl = renderer.getContext();
const ext = gl.getExtension("WEBGL_debug_renderer_info");
if (ext) {
  console.log("GPU:", gl.getParameter(ext.UNMASKED_RENDERER_WEBGL));
}

// Count active GPU resources via Three.js renderer info
const info = renderer.info;
console.log(`Geometries: ${info.memory.geometries}`);
console.log(`Textures: ${info.memory.textures}`);
console.log(`Programs: ${info.programs?.length}`);
console.log(`Draw calls: ${info.render.calls}`);
console.log(`Triangles: ${info.render.triangles}`);
```

## FragmentsManager State Properties

| Property | Type | Purpose |
|---|---|---|
| `list` | `Map<string, FragmentsModel>` | All loaded models |
| `initialized` | `boolean` | Whether worker is ready |
| `baseCoordinationModel` | `string` | ID of the base model |
| `baseCoordinationMatrix` | `THREE.Matrix4` | Base model's world matrix |

## FragmentsManager Events

| Event | Payload | When |
|---|---|---|
| `onFragmentsLoaded` | `FragmentsModel` | After a model finishes loading |
| `onBeforeDispose` | `FragmentsModel` | Before a model is disposed |
| `onDisposed` | `void` | After FragmentsManager itself is disposed |

## BVH Raycasting

Applied automatically in the `Components` constructor via `three-mesh-bvh`.
Patches `THREE.BufferGeometry.prototype` to support BVH acceleration.

**No API surface — fully automatic.** The patch adds:
- `boundsTree` property on BufferGeometry
- Modified `Raycaster.intersectObject` that uses BVH when available

Performance: O(log n) intersection tests instead of O(n).

## Polygon Offset Properties

```typescript
// Available on all Three.js Material subclasses
material.polygonOffset: boolean;       // Enable offset (default: false)
material.polygonOffsetFactor: number;  // Scale factor (default: 0)
material.polygonOffsetUnits: number;   // Constant offset (default: 0)
```

Standard values for overlay prevention:
- `polygonOffset = true`
- `polygonOffsetFactor = 1`
- `polygonOffsetUnits = 1`

Negative values push surfaces closer to camera. Positive values push away.
