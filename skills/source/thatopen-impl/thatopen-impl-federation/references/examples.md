# Federation Examples

## Example 1: Multi-Model Load with Coordination

Load three discipline models and align them to a shared coordinate system.

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

const components = new OBC.Components();

// World setup
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.camera = new OBC.OrthoPerspectiveCamera(components);

// Fragment and loader setup
const fragments = components.get(OBC.FragmentsManager);
fragments.init("https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs");

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

// Load architectural model as base
const archModel = await ifcLoader.load(archBytes, true, "Architectural");
fragments.baseCoordinationModel = archModel.modelId;
fragments.baseCoordinationMatrix = archModel.coordinationMatrix;

// Load structural model — aligned automatically (coordinate=true)
const structModel = await ifcLoader.load(structBytes, true, "Structural");

// Load MEP model — aligned automatically
const mepModel = await ifcLoader.load(mepBytes, true, "MEP");

// Fit camera to encompass all models
const boxer = components.get(OBC.BoundingBoxer);
boxer.addFromModels();
const camera = world.camera as OBC.OrthoPerspectiveCamera;
camera.fit([boxer.get()]);

console.log(`Loaded ${fragments.list.size} models`);
```

---

## Example 2: Per-Model Visibility Toggle

Create a UI-driven model visibility toggle using Hider and Classifier.

```typescript
const hider = components.get(OBC.Hider);
const classifier = components.get(OBC.Classifier);

// Classify by model after loading
classifier.byModel();

// Toggle function for UI buttons
async function setModelVisibility(modelName: string, visible: boolean) {
  const items = await classifier.find({ models: [modelName] });
  hider.set(visible, items);
}

// Usage
await setModelVisibility("Structural", false); // hide structural
await setModelVisibility("Structural", true);  // show structural
```

---

## Example 3: Isolate a Single Model

Show only one discipline, hiding everything else.

```typescript
const hider = components.get(OBC.Hider);
const classifier = components.get(OBC.Classifier);
classifier.byModel();

// Isolate MEP — hides architectural and structural
const mepItems = await classifier.find({ models: ["MEP"] });
hider.isolate(mepItems);

// Later, show everything again
hider.set(true);
```

---

## Example 4: Hide Category Across All Models

Hide all walls regardless of which model they belong to.

```typescript
const hider = components.get(OBC.Hider);
const classifier = components.get(OBC.Classifier);
classifier.byCategory();

// Find all walls across all models
const allWalls = await classifier.find({ categories: ["IFCWALL"] });
hider.set(false, allWalls);

// Show them again
hider.set(true, allWalls);
```

---

## Example 5: BoundingBoxer — Fit Camera to Selection

Compute a bounding box for a subset of elements and fit the camera.

```typescript
const boxer = components.get(OBC.BoundingBoxer);
const classifier = components.get(OBC.Classifier);
classifier.byModel();
classifier.byIfcBuildingStorey();

// Get ground floor elements across all models
const groundFloor = await classifier.find({ storeys: ["Ground Floor"] });

// Compute bounding box and fit camera
boxer.addFromModelIdMap(groundFloor);
const box = boxer.get();

const camera = world.camera as OBC.OrthoPerspectiveCamera;
camera.fit([box]);
```

---

## Example 6: Camera Orientation for Architectural Views

Use BoundingBoxer to position the camera for standard views.

```typescript
const boxer = components.get(OBC.BoundingBoxer);
boxer.addFromModels(); // include all models

// Front elevation
const front = boxer.getCameraOrientation("front", 1.5);
camera.controls.setLookAt(
  front.position.x, front.position.y, front.position.z,
  front.target.x, front.target.y, front.target.z,
  true
);

// Top-down plan view
const top = boxer.getCameraOrientation("top", 2.0);
camera.controls.setLookAt(
  top.position.x, top.position.y, top.position.z,
  top.target.x, top.target.y, top.target.z,
  true
);
```

---

## Example 7: Cross-Model Query and Highlight

Find elements across models and highlight them.

```typescript
const classifier = components.get(OBC.Classifier);
const fragments = components.get(OBC.FragmentsManager);
classifier.byModel();
classifier.byCategory();

// Find walls in architectural + columns in structural
const archWalls = await classifier.find({
  models: ["Architectural"],
  categories: ["IFCWALL"]
});
const structColumns = await classifier.find({
  models: ["Structural"],
  categories: ["IFCCOLUMN"]
});

// Merge into one ModelIdMap
const combined: Record<string, Set<number>> = { ...archWalls };
for (const [modelId, ids] of Object.entries(structColumns)) {
  if (combined[modelId]) {
    for (const id of ids) combined[modelId].add(id);
  } else {
    combined[modelId] = new Set(ids);
  }
}

// Highlight the combined selection
await fragments.highlight(
  { color: new THREE.Color("#FF6600"), opacity: 0.7 },
  combined
);
```

---

## Example 8: Model Load Event and Auto-Classify

Automatically classify every model as it loads.

```typescript
const fragments = components.get(OBC.FragmentsManager);
const classifier = components.get(OBC.Classifier);

fragments.onFragmentsLoaded.add((model) => {
  console.log(`Model loaded: ${model.name}`);

  // Re-classify to include the new model
  classifier.byModel();
  classifier.byCategory();
  classifier.byIfcBuildingStorey();
});
```

---

## Example 9: Selective Model Disposal

Remove one model from a federated scene without affecting others.

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Check loaded models
console.log("Before:", [...fragments.list.keys()]);

// Dispose only the MEP model
fragments.disposeModel(mepModel.modelId);

console.log("After:", [...fragments.list.keys()]);
// MEP model is gone; architectural and structural remain
```

---

## Example 10: Get Visibility State

Query which elements are currently visible or hidden.

```typescript
const hider = components.get(OBC.Hider);

// Hide some elements first
hider.set(false, someItems);

// Query visible elements across all models
const visibleMap = hider.getVisibilityMap(true);
for (const [modelId, ids] of visibleMap) {
  console.log(`Model ${modelId}: ${ids.size} visible elements`);
}

// Query hidden elements in a specific model only
const hiddenInArch = hider.getVisibilityMap(false, [archModel.modelId]);
console.log(`Hidden in arch: ${hiddenInArch.get(archModel.modelId)?.size ?? 0}`);
```
