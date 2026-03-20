# Loading Errors — Error Types, Messages & Fix Patterns

## Error Classification

### Category 1: WASM Initialization Errors

These errors occur before any IFC file is opened. They indicate the WASM engine itself failed to start.

| Error Type | Message Pattern | Thrown By | Fix |
|---|---|---|---|
| `CompileError` | `WebAssembly.instantiate(): expected magic word 00 61 73 6d` | Browser WASM engine | WASM file corrupt or wrong MIME type. Serve as `application/wasm`. |
| `CompileError` | `WebAssembly.instantiate() failed` | Browser WASM engine | WASM file truncated or wrong version. Re-download or match version. |
| `RuntimeError` | `unreachable` | web-ifc WASM | Version mismatch between JS API and WASM binary. |
| `TypeError` | `Cannot read properties of undefined (reading '...')` | IfcLoader | `setup()` not called — WASM not initialized. |
| `Error` | `Failed to fetch` on `.wasm` URL | fetch API | WASM path wrong, file missing, or network error. |
| `Error` | `404 Not Found` | HTTP server | WASM path incorrect or files not deployed. |

### Category 2: IFC Parse Errors

These errors occur when opening or parsing an IFC file after WASM is initialized.

| Error Type | Message Pattern | Thrown By | Fix |
|---|---|---|---|
| `Error` | `Invalid IFC file` | web-ifc | File is not valid STEP-encoded IFC. Verify file header. |
| `Error` | `Unexpected token` or parser errors | web-ifc | Corrupt IFC file or encoding issue. Use `arrayBuffer()` not `text()`. |
| `RuntimeError` | `memory access out of bounds` | WASM runtime | Model too large for WASM memory. Set `MEMORY_LIMIT` or reduce model. |
| `Error` | `Boolean operation timeout` | web-ifc | Complex geometry. Set `BOOL_ABORT_THRESHOLD`. |
| `TypeError` | `data is not Uint8Array` | IfcLoader | Wrong data type. Convert with `new Uint8Array(buffer)`. |

### Category 3: Worker Errors

These errors relate to the Web Worker used by FragmentsManager.

| Error Type | Message Pattern | Thrown By | Fix |
|---|---|---|---|
| `Error` | `FragmentsManager not initialized` | FragmentsManager | Call `fragmentsManager.init(workerURL)` before loading. |
| `DOMException` | `Failed to construct 'Worker'` | Browser Worker API | Worker URL invalid or blocked by CSP. |
| `Error` | `404` on worker script | HTTP server | Worker URL path incorrect. |
| `SecurityError` | CORS error on worker | Browser | Worker served from different origin without CORS headers. |

### Category 4: Rendering Errors (Post-Load)

Model loads without error but visual problems occur.

| Symptom | Root Cause | Fix |
|---|---|---|
| Black / empty viewport | `components.init()` not called | Call `components.init()` to start render loop. |
| Nothing visible | `model.object` not added to scene | Add with `world.scene.three.add(model.object)`. |
| Model invisible but loaded | Camera position far from model | Call `world.camera.fit(world.scene.three.children)`. |
| Container shows nothing | Container has 0px height | Set CSS: `height: 100vh` or explicit pixel height. |
| Z-fighting / flickering | Large world coordinates | Use `COORDINATE_TO_ORIGIN: true`. |
| Geometry distorted | Floating-point precision loss | Use `COORDINATE_TO_ORIGIN: true`. |
| Missing elements | Excluded IFC categories | Add classes via `onIfcImporterInitialized`. |

---

## Error Detection Methods

### Checking WASM Initialization State

```typescript
// IfcLoader: check isSetup
const ifcLoader = components.get(OBC.IfcLoader);
console.log("IfcLoader setup:", ifcLoader.isSetup); // boolean

// FragmentsManager: check initialized
const fm = components.get(OBC.FragmentsManager);
console.log("FragmentsManager initialized:", fm.initialized); // boolean
```

### Checking Model Load Success

```typescript
const model = await ifcLoader.load(data, true, name);

// Verify model has content
console.log("Model has geometry:", model.object.children.length > 0);

// Verify model bounding box
const box = new THREE.Box3().setFromObject(model.object);
console.log("Model size:", box.getSize(new THREE.Vector3()));
```

### Checking WASM File Availability

```typescript
// Pre-check WASM file before initializing
async function checkWasmAvailable(wasmPath: string): Promise<boolean> {
  try {
    const response = await fetch(wasmPath + "web-ifc.wasm", { method: "HEAD" });
    const contentType = response.headers.get("content-type");
    if (!response.ok) {
      console.error(`WASM file not found: ${response.status}`);
      return false;
    }
    if (contentType && !contentType.includes("wasm")) {
      console.warn(`WASM served with wrong MIME: ${contentType}`);
    }
    return true;
  } catch (e) {
    console.error("WASM file unreachable:", e.message);
    return false;
  }
}
```

---

## web-ifc LogLevel Settings

Control web-ifc logging verbosity for debugging:

```typescript
import { LogLevel } from "web-ifc";

// During development — see all messages
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_DEBUG);

// For troubleshooting — errors only
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_ERROR);

// Production — silent
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_OFF);
```

When debugging loading issues, ALWAYS set `LOG_LEVEL_DEBUG` temporarily to capture web-ifc's internal messages.

---

## LoaderSettings for Error Prevention

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;    // Prevents E-09 (large coordinate issues)
  USE_FAST_BOOLS?: boolean;          // Prevents E-12 (boolean hangs) — less accurate
  BOOL_ABORT_THRESHOLD?: number;     // Prevents E-12 (timeout in ms)
  MEMORY_LIMIT?: number;             // Prevents E-14 (memory exhaustion, in bytes)
  CIRCLE_SEGMENTS_LOW?: number;      // Reduces geometry detail for performance
  CIRCLE_SEGMENTS_MEDIUM?: number;
  CIRCLE_SEGMENTS_HIGH?: number;
}
```

### Defensive Loading Configuration

```typescript
await ifcLoader.setup();
ifcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;
ifcLoader.settings.webIfc.BOOL_ABORT_THRESHOLD = 10000;
ifcLoader.settings.webIfc.MEMORY_LIMIT = 2147483648; // 2GB
```

This configuration prevents the most common loading failures for unknown IFC files.
