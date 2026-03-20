# IfcLoader & IfcImporter — Working Examples

## Example 1: Basic IFC Loading

The minimal setup for loading an IFC file into a ThatOpen viewer.

```typescript
import * as OBC from "@thatopen/components";

// 1. Create component framework
const components = new OBC.Components();

// 2. Create world with scene, camera, renderer
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// 3. Initialize FragmentsManager
const fragmentsManager = components.get(OBC.FragmentsManager);
fragmentsManager.init(workerURL);

// 4. Setup IfcLoader (autoSetWasm is true by default)
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// 5. Start rendering
components.init();

// 6. Load IFC file
const response = await fetch("/models/building.ifc");
const buffer = await response.arrayBuffer();
const data = new Uint8Array(buffer);

const model = await ifcLoader.load(data, true, "Building");
world.scene.three.add(model.object);

// 7. Frame camera to fit model
world.camera.fit([model.object]);
```

---

## Example 2: Custom WASM Path (CDN)

When `autoSetWasm` does not work (e.g., bundler issues) or you want explicit control.

```typescript
const ifcLoader = components.get(OBC.IfcLoader);

await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.77/",
    absolute: true,
  },
});
```

NEVER hardcode the version without checking `package.json`. To stay in sync:

```typescript
// Read version from package.json or define as a constant
import { version as webIfcVersion } from "web-ifc/package.json";

await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: `https://unpkg.com/web-ifc@${webIfcVersion}/`,
    absolute: true,
  },
});
```

---

## Example 3: Custom WASM Path (Local Files)

For offline/air-gapped deployments, serve WASM files from your own static directory.

```typescript
// Copy from node_modules/web-ifc/:
//   web-ifc.wasm
//   web-ifc-mt.wasm (for multi-threaded mode)
// to your public/static/wasm/ directory

await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "/static/wasm/",
    absolute: false,
  },
});
```

---

## Example 4: Configuring web-ifc LoaderSettings

Apply web-ifc settings for coordinate origin translation and faster boolean operations.

```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// Configure before calling load()
ifcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
ifcLoader.settings.webIfc.USE_FAST_BOOLS = true;

const model = await ifcLoader.load(data, true, "GeoModel");
```

---

## Example 5: Customizing Imported IFC Classes

Control which IFC entity types are converted to fragments.

```typescript
import * as WEBIFC from "web-ifc";

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// Modify classes when the importer initializes
ifcLoader.onIfcImporterInitialized.add((importer) => {
  // Add spaces (excluded by default)
  importer.classes.elements.add(WEBIFC.IFCSPACE);

  // Add openings (excluded by default)
  importer.classes.elements.add(WEBIFC.IFCOPENINGELEMENT);

  // Remove furniture to save memory
  importer.classes.elements.delete(WEBIFC.IFCFURNISHINGELEMENT);

  // Skip items far from origin (georeferenced models)
  importer.distanceThreshold = 50000;
});

const model = await ifcLoader.load(data, true, "CustomImport");
```

---

## Example 6: IfcImporter (Low-Level, Without Components)

Use IfcImporter directly for server-side or Web Worker IFC conversion.

```typescript
import { IfcImporter } from "@thatopen/fragments";

const importer = new IfcImporter();

// Configure WASM (required)
importer.wasm.path = "https://unpkg.com/web-ifc@0.0.77/";
importer.wasm.absolute = true;

// Optional: customize what gets imported
importer.classes.elements.add(WEBIFC.IFCSPACE);
importer.distanceThreshold = null;  // no distance filtering

// Convert IFC → Fragments binary
const ifcData = new Uint8Array(await fetch("/model.ifc").then(r => r.arrayBuffer()));
const fragmentsData = await importer.process(ifcData);

// Store fragmentsData (Uint8Array) for later loading via FragmentsManager
await saveToIndexedDB("model.frg", fragmentsData);
```

---

## Example 7: Convert Once, Reload Fast (Fragment Caching)

The recommended production workflow to avoid re-parsing IFC on every page load.

```typescript
const fragmentsManager = components.get(OBC.FragmentsManager);
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

async function loadModel(ifcUrl: string, cacheKey: string) {
  // Try loading cached fragments first
  const cached = await loadFromIndexedDB(cacheKey);
  if (cached) {
    const model = fragmentsManager.load(cached);
    world.scene.three.add(model.object);
    return model;
  }

  // No cache: convert IFC → Fragments
  const response = await fetch(ifcUrl);
  const data = new Uint8Array(await response.arrayBuffer());
  const model = await ifcLoader.load(data, true, cacheKey);
  world.scene.three.add(model.object);

  // Cache the fragments binary for next time
  const fragmentsData = model.export();
  await saveToIndexedDB(cacheKey, fragmentsData);

  return model;
}
```

---

## Example 8: Loading Multiple Models with Coordination

Load multiple IFC files that align spatially using the coordination matrix.

```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

const files = [
  { url: "/models/architecture.ifc", name: "Architecture" },
  { url: "/models/structure.ifc", name: "Structure" },
  { url: "/models/mep.ifc", name: "MEP" },
];

for (const file of files) {
  const response = await fetch(file.url);
  const data = new Uint8Array(await response.arrayBuffer());

  // coordinate: true ensures all models align to the same origin
  const model = await ifcLoader.load(data, true, file.name);
  world.scene.three.add(model.object);
}
```

---

## Example 9: Using readIfcFile for Raw Access

When you need web-ifc queries without full fragment conversion.

```typescript
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

const data = new Uint8Array(await fetch("/model.ifc").then(r => r.arrayBuffer()));
const modelID = await ifcLoader.readIfcFile(data);

// Direct web-ifc access
const schema = ifcLoader.webIfc.GetModelSchema(modelID);
console.log("IFC Schema:", schema);  // "IFC2X3", "IFC4", or "IFC4X3"

const wallIDs = ifcLoader.webIfc.GetLineIDsWithType(modelID, WEBIFC.IFCWALL);
console.log("Number of walls:", wallIDs.size());

// ALWAYS clean up after raw access
ifcLoader.cleanUp();
```
