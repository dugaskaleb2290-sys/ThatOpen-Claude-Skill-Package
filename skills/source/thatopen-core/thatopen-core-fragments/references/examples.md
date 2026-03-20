# Fragment System Examples

## 1. Initialize Worker

ALWAYS initialize the worker before any fragment operations.

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();

// Get FragmentsManager (lazy singleton)
const fragments = components.get(OBC.FragmentsManager);

// Initialize worker — MUST match installed @thatopen/fragments version
fragments.init(
  "https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs"
);

// Verify initialization
console.log(fragments.initialized); // true
```

**Local worker alternative:**

```typescript
// If bundling the worker locally (e.g., copied to public/):
fragments.init("/workers/fragments-worker.mjs");
```

---

## 2. Load IFC and Convert to Fragments

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBC.SimpleRenderer
>();

world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);
components.init();

// Initialize fragments worker
const fragments = components.get(OBC.FragmentsManager);
fragments.init(workerURL);

// Setup IFC loader (configures WASM)
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// Load IFC file
const response = await fetch("/models/building.ifc");
const data = new Uint8Array(await response.arrayBuffer());
const model = await ifcLoader.load(data, true, "MyBuilding");

// model is now a FragmentsModel added to the scene
console.log("Model ID:", model.modelId);
console.log("Loaded models:", fragments.list.size);
```

---

## 3. getData — Extract Element Properties

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Target specific elements
const items = {
  [model.modelId]: new Set([42, 43, 44])
};

// Extract IFC property data (runs in worker)
const data = await fragments.getData(items);

// Process results
for (const [modelId, itemDataArray] of Object.entries(data)) {
  for (const itemData of itemDataArray) {
    console.log("Element:", itemData);
  }
}
```

---

## 4. Raycast — Pick Elements in 3D

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Normalized mouse coordinates from a click event
function onPointerClick(event: PointerEvent) {
  const rect = renderer.three.domElement.getBoundingClientRect();
  const mouse = new THREE.Vector2(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1
  );

  fragments.raycast({
    camera: world.camera.three,
    mouse,
    dom: renderer.three.domElement
  }).then((result) => {
    if (result) {
      console.log("Hit model:", result.modelId);
      console.log("Hit element:", result.localId);
      console.log("Hit point:", result.point);

      // Highlight the hit element
      const hitItems = {
        [result.modelId]: new Set([result.localId])
      };
      fragments.highlight(
        { color: new THREE.Color("#ff0000"), opacity: 0.8 },
        hitItems
      );
    }
  });
}
```

**With snapping classes (restrict to specific IFC types):**

```typescript
import { IFCWALL, IFCSLAB, IFCCOLUMN } from "web-ifc";

const result = await fragments.raycast({
  camera: world.camera.three,
  mouse,
  dom: renderer.three.domElement,
  snappingClasses: [IFCWALL, IFCSLAB, IFCCOLUMN]
});
```

---

## 5. Coordinate Alignment for Multi-Model Federation

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Load first model — becomes the base
const modelA = await ifcLoader.load(ifcBytesA, true, "Building-A");
fragments.baseCoordinationModel = modelA.modelId;
fragments.baseCoordinationMatrix = modelA.coordinationMatrix;

// Load second model — align to base
const modelB = await ifcLoader.load(ifcBytesB, true, "Building-B");
fragments.applyBaseCoordinateSystem(modelB, modelB.coordinationMatrix);

// Load third model — also align to base
const modelC = await ifcLoader.load(ifcBytesC, true, "MEP-Model");
fragments.applyBaseCoordinateSystem(modelC, modelC.coordinationMatrix);

// All three models now share the same coordinate origin
```

---

## 6. GUID Mapping

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Convert IFC GlobalIds to ModelIdMap for targeting
const guids = ["2O2Fr$t4X7Zf8NOew3FLOH", "3MVF2h$Kv7gPC5cv0IKXCA"];
const items = await fragments.guidsToModelIdMap(guids);

// Use the items map for any operation
await fragments.highlight(
  { color: new THREE.Color("#00ff00"), opacity: 0.5 },
  items
);

// Convert back to GUIDs (e.g., for BCF export)
const exportedGuids = await fragments.modelIdMapToGuids(items);
console.log(exportedGuids); // ["2O2Fr$t4X7Zf8NOew3FLOH", "3MVF2h$Kv7gPC5cv0IKXCA"]
```

---

## 7. Get Positions and Bounding Boxes

```typescript
const fragments = components.get(OBC.FragmentsManager);

const items = { [model.modelId]: new Set([42, 43]) };

// Get center positions
const positions = await fragments.getPositions(items);
positions.forEach((pos, i) => {
  console.log(`Element ${i}: x=${pos.x}, y=${pos.y}, z=${pos.z}`);
});

// Get bounding boxes
const boxes = await fragments.getBBoxes(items);
boxes.forEach((box, i) => {
  console.log(`Element ${i}: min=${box.min.toArray()}, max=${box.max.toArray()}`);
});
```

---

## 8. Highlight and Reset

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Define styles
const selectionStyle = { color: new THREE.Color("#BCF124"), opacity: 0.6 };
const errorStyle = { color: new THREE.Color("#ff0000"), opacity: 0.8 };

// Highlight a selection
const selectedItems = { [model.modelId]: new Set([10, 20, 30]) };
await fragments.highlight(selectionStyle, selectedItems);

// Highlight errors differently
const errorItems = { [model.modelId]: new Set([99]) };
await fragments.highlight(errorStyle, errorItems);

// Reset only the selection (errors stay highlighted)
await fragments.resetHighlight(selectedItems);

// Reset all highlights
await fragments.resetHighlight();
```

---

## 9. Listen to Fragment Events

```typescript
const fragments = components.get(OBC.FragmentsManager);

// React to model loading
fragments.onFragmentsLoaded.add((model) => {
  console.log("Model loaded:", model.modelId);
  console.log("Fragments count:", model.fragments.length);
});

// React before disposal (e.g., save state)
fragments.onBeforeDispose.add((model) => {
  console.log("About to dispose:", model.modelId);
});

// React after full disposal
fragments.onDisposed.add(() => {
  console.log("FragmentsManager fully disposed");
});
```

---

## 10. Dispose a Single Model

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Dispose one model (frees GPU memory, removes from scene)
fragments.disposeModel(model.modelId);

// Verify removal
console.log(fragments.list.has(model.modelId)); // false

// Dispose everything when leaving the page
components.dispose();
```
