# Federation Anti-Patterns

## AP-1: Missing Coordinate Alignment

**Wrong:**
```typescript
const archModel = await ifcLoader.load(archBytes, true, "Architectural");
const structModel = await ifcLoader.load(structBytes, true, "Structural");
// Models appear at different positions — no coordination set
```

**Why it fails:** Each IFC file contains a coordination matrix that positions
the model in world space. Without setting a base and aligning subsequent
models, they appear at their original authoring origins, which may be hundreds
of meters apart.

**Correct:**
```typescript
const archModel = await ifcLoader.load(archBytes, true, "Architectural");
fragments.baseCoordinationModel = archModel.modelId;
fragments.baseCoordinationMatrix = archModel.coordinationMatrix;

const structModel = await ifcLoader.load(structBytes, true, "Structural");
// With coordinate=true and base set, alignment happens automatically
```

**Rule:** ALWAYS set `baseCoordinationModel` and `baseCoordinationMatrix` from
the first loaded model before loading any additional models.

---

## AP-2: Using expressID Instead of ModelIdMap

**Wrong:**
```typescript
// Trying to hide element 42 — but which model?
hider.set(false, { "": new Set([42]) });
```

**Why it fails:** Express IDs are only unique within a single IFC model. Two
models may both have an element with express ID 42 referring to completely
different objects. Using an empty string or wrong model ID causes silent
failures or affects the wrong elements.

**Correct:**
```typescript
const items: ModelIdMap = {
  [archModel.modelId]: new Set([42])
};
hider.set(false, items);
```

**Rule:** ALWAYS use the correct model UUID as the key in ModelIdMap. NEVER
use express IDs without their model context.

---

## AP-3: Manual Hide-All-Then-Show Instead of Isolate

**Wrong:**
```typescript
// Hide everything, then show one model
hider.set(false);
const mepItems = await classifier.find({ models: ["MEP"] });
hider.set(true, mepItems);
// Two separate operations — may cause visual flicker
```

**Why it fails:** Two separate visibility calls create a frame where everything
is hidden, causing a visual flash. The state between calls is inconsistent.

**Correct:**
```typescript
const mepItems = await classifier.find({ models: ["MEP"] });
hider.isolate(mepItems);
// Atomic operation — no flicker
```

**Rule:** ALWAYS use `hider.isolate()` when you want to show only specific
elements. NEVER implement isolation manually with sequential set() calls.

---

## AP-4: Forgetting to Classify Before Querying

**Wrong:**
```typescript
const classifier = components.get(OBC.Classifier);
// No classification calls made
const items = await classifier.find({ models: ["Architectural"] });
// Returns empty — classifier has no data
```

**Why it fails:** `classifier.find()` queries the internal classification
lists. These lists are empty until you call `byModel()`, `byCategory()`, or
`byIfcBuildingStorey()`. The classifier does not auto-populate.

**Correct:**
```typescript
const classifier = components.get(OBC.Classifier);
classifier.byModel();
classifier.byCategory();
const items = await classifier.find({ models: ["Architectural"] });
```

**Rule:** ALWAYS call the relevant `by*()` methods before using
`classifier.find()`. Call them again after loading new models.

---

## AP-5: Not Re-Classifying After Loading New Models

**Wrong:**
```typescript
classifier.byModel();
// Load model A — classified
const modelA = await ifcLoader.load(bytesA, true, "ModelA");

// Load model B — NOT in classifier yet
const modelB = await ifcLoader.load(bytesB, true, "ModelB");

const bItems = await classifier.find({ models: ["ModelB"] });
// Returns empty — ModelB was loaded after classification
```

**Why it fails:** Classification is a snapshot operation. It groups elements
that are loaded at the time of the call. Models loaded after `byModel()` are
not included in the classification.

**Correct:**
```typescript
const modelA = await ifcLoader.load(bytesA, true, "ModelA");
const modelB = await ifcLoader.load(bytesB, true, "ModelB");
classifier.byModel(); // classify AFTER all models are loaded

// OR re-classify after each load
fragments.onFragmentsLoaded.add(() => {
  classifier.byModel();
  classifier.byCategory();
});
```

**Rule:** ALWAYS re-classify after loading new models. Use the
`onFragmentsLoaded` event for automatic re-classification.

---

## AP-6: Calling BoundingBoxer.get() Without Adding Items

**Wrong:**
```typescript
const boxer = components.get(OBC.BoundingBoxer);
const box = boxer.get();
// Returns an empty or invalid Box3 — nothing was added
```

**Why it fails:** BoundingBoxer accumulates items through `addFromModels()` or
`addFromModelIdMap()`. Calling `get()` without adding anything returns an
uninitialized bounding box.

**Correct:**
```typescript
const boxer = components.get(OBC.BoundingBoxer);
boxer.addFromModels(); // or boxer.addFromModelIdMap(items)
const box = boxer.get();
```

**Rule:** ALWAYS call `addFromModels()` or `addFromModelIdMap()` before
`get()` or `getCameraOrientation()`.

---

## AP-7: Disposing Models Without Updating Classification

**Wrong:**
```typescript
fragments.disposeModel(mepModel.modelId);
// classifier still has stale entries for the disposed model
const mepItems = await classifier.find({ models: ["MEP"] });
hider.set(false, mepItems);
// Operates on stale data — may cause errors or no-ops
```

**Why it fails:** Disposing a model removes it from `fragments.list` and frees
GPU resources, but the classifier retains its stale classification entries.
Operating on these stale entries targets elements that no longer exist.

**Correct:**
```typescript
fragments.disposeModel(mepModel.modelId);
// Re-classify to remove stale entries
classifier.byModel();
classifier.byCategory();
```

**Rule:** ALWAYS re-classify after disposing models to keep classification
data in sync with loaded models.

---

## AP-8: Setting Coordination Base After Loading Multiple Models

**Wrong:**
```typescript
const archModel = await ifcLoader.load(archBytes, true, "Architectural");
const structModel = await ifcLoader.load(structBytes, true, "Structural");

// Too late — structural model was already placed without coordination
fragments.baseCoordinationModel = archModel.modelId;
fragments.baseCoordinationMatrix = archModel.coordinationMatrix;
```

**Why it fails:** The coordination base must be set before loading subsequent
models. When `coordinate=true` is passed to `load()`, the loader uses the
current base to align the model. If no base is set, alignment is skipped.

**Correct:**
```typescript
const archModel = await ifcLoader.load(archBytes, true, "Architectural");
fragments.baseCoordinationModel = archModel.modelId;
fragments.baseCoordinationMatrix = archModel.coordinationMatrix;
// NOW load additional models
const structModel = await ifcLoader.load(structBytes, true, "Structural");
```

**Rule:** ALWAYS set the coordination base immediately after loading the first
model, before loading any subsequent models.
