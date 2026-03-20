# Model Analyzer — Anti-Patterns

## AP-1: Querying All Properties Without Filtering

**Wrong:**
```typescript
// Fetching properties for EVERY element in the model
const allItems: ModelIdMap = {};
for (const [modelId, model] of fragments.list) {
  const allIDs = ifcApi.GetAllLines(modelID);
  const ids = new Set<number>();
  for (let i = 0; i < allIDs.size(); i++) {
    ids.add(allIDs.get(i));
  }
  allItems[modelId] = ids;
}
const data = await fragments.getData(allItems); // Blocks main thread
```

**Why it fails:** `getData()` serializes results from the worker back to
the main thread. For models with 50,000+ entities, this transfer causes
severe jank or out-of-memory errors.

**Correct:**
```typescript
// Filter by type first, then batch
await classifier.byCategory();
const wallItems = await classifier.find({ Categories: ["IFCWALL"] });
const wallData = await fragments.getData(wallItems);
// Repeat for other types as needed
```

---

## AP-2: Classifying Before Loading

**Wrong:**
```typescript
const classifier = components.get(OBC.Classifier);
await classifier.byCategory();  // No models loaded yet!
// ... later ...
const model = await loader.load(data, true, "model.ifc");
// classifier.list is empty — classification ran on nothing
```

**Why it fails:** Classification methods read from loaded FragmentsModels.
Running them before any model is loaded produces empty groups.

**Correct:**
```typescript
const model = await loader.load(data, true, "model.ifc");
// THEN classify
await classifier.byCategory();
await classifier.byIfcBuildingStorey();
```

---

## AP-3: Using GetLine with flatten:true for Bulk Queries

**Wrong:**
```typescript
const allIDs = ifcApi.GetAllLines(modelID);
for (let i = 0; i < allIDs.size(); i++) {
  const entity = ifcApi.GetLine(modelID, allIDs.get(i), true); // flatten=true
  // Process...
}
```

**Why it fails:** `flatten: true` recursively resolves every reference in
the entity. For a model with 100,000 entities, this creates millions of
recursive calls and causes exponential slowdown or stack overflow.

**Correct:**
```typescript
// Use flatten:false (default) for bulk queries
for (let i = 0; i < allIDs.size(); i++) {
  const entity = ifcApi.GetLine(modelID, allIDs.get(i));
  // Resolve specific references manually only when needed
}

// Or use GetRawLineData for statistics (faster, no parsing)
for (let i = 0; i < allIDs.size(); i++) {
  const raw = ifcApi.GetRawLineData(modelID, allIDs.get(i));
  // raw.type gives the IFC type code without parsing properties
}
```

---

## AP-4: Missing FragmentsManager Initialization

**Wrong:**
```typescript
const fragments = components.get(OBC.FragmentsManager);
const data = await fragments.getData(items); // Throws or fails silently
```

**Why it fails:** `getData()` delegates to a web worker. Without calling
`fragments.init(workerURL)` first, the worker is not available.

**Correct:**
```typescript
const fragments = components.get(OBC.FragmentsManager);
fragments.init(workerURL); // MUST be called once before any getData()
// ... later ...
const data = await fragments.getData(items);
```

---

## AP-5: Ignoring Vector Access Pattern for web-ifc Results

**Wrong:**
```typescript
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
for (let i = 0; i < wallIDs.length; i++) {  // .length does not exist!
  const wall = ifcApi.GetLine(modelID, wallIDs[i]); // [] indexing fails!
}
```

**Why it fails:** web-ifc returns `Vector<number>`, a WASM type. It does
NOT have `.length` or `[]` indexing. ALWAYS use `.size()` and `.get(i)`.

**Correct:**
```typescript
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
for (let i = 0; i < wallIDs.size(); i++) {
  const wall = ifcApi.GetLine(modelID, wallIDs.get(i));
}
```

---

## AP-6: Not Handling Missing Property Sets

**Wrong:**
```typescript
const psets = await ifcApi.properties.getPropertySets(modelID, expressID);
for (const pset of psets) {  // Crashes if psets is null/undefined
  console.log(pset.Name.value);  // Crashes if Name is missing
}
```

**Why it fails:** Not all elements have property sets. Not all property sets
have all expected fields. IFC models from different authoring tools vary
significantly in completeness.

**Correct:**
```typescript
const psets = await ifcApi.properties.getPropertySets(modelID, expressID);
if (psets && psets.length > 0) {
  for (const pset of psets) {
    const name = pset.Name?.value ?? "Unnamed";
    console.log(name);
  }
} else {
  console.log(`Element #${expressID}: no property sets`);
}
```

---

## AP-7: Re-running Classification for Every Query

**Wrong:**
```typescript
// In a loop or repeated function
for (const type of typesToAnalyze) {
  await classifier.byCategory();  // Redundant! Rebuilds entire index
  const items = await classifier.find({ Categories: [type] });
  // Process items...
}
```

**Why it fails:** `byCategory()` reads the entire model and rebuilds the
classification index. Calling it repeatedly wastes time. The classification
persists in `classifier.list` until the model is disposed.

**Correct:**
```typescript
// Classify ONCE
await classifier.byCategory();

// Query multiple times from cached classification
for (const type of typesToAnalyze) {
  const items = await classifier.find({ Categories: [type] });
  // Process items...
}
```

---

## AP-8: Creating Multiple IfcAPI Instances for Analysis

**Wrong:**
```typescript
import * as WebIFC from "web-ifc";
const myApi = new WebIFC.IfcAPI(); // Second instance!
await myApi.Init();
const myModelID = myApi.OpenModel(data);
// Now two WASM modules loaded — doubled memory usage
```

**Why it fails:** Each `IfcAPI` instance loads its own WASM module. The
IfcLoader component already has a `webIfc` property with the initialized
instance. Creating a second one wastes memory.

**Correct:**
```typescript
const loader = components.get(OBC.IfcLoader);
const ifcApi = loader.webIfc; // Reuse the existing instance
```

---

## AP-9: Incomplete Spatial Hierarchy Validation

**Wrong:**
```typescript
// Only checking if IfcBuildingStorey exists
const storeyIDs = ifcApi.GetLineIDsWithType(modelID, IFCBUILDINGSTOREY);
if (storeyIDs.size() > 0) {
  console.log("Model has valid spatial structure"); // Not necessarily true
}
```

**Why it fails:** A valid IFC spatial hierarchy requires IfcProject >
IfcSite > IfcBuilding > IfcBuildingStorey. Having storeys without a proper
parent chain indicates a malformed model.

**Correct:**
```typescript
import {
  IFCPROJECT, IFCSITE, IFCBUILDING, IFCBUILDINGSTOREY
} from "web-ifc";

const requiredLevels = [
  { type: IFCPROJECT, name: "IfcProject" },
  { type: IFCSITE, name: "IfcSite" },
  { type: IFCBUILDING, name: "IfcBuilding" },
  { type: IFCBUILDINGSTOREY, name: "IfcBuildingStorey" },
];

let hierarchyValid = true;
for (const { type, name } of requiredLevels) {
  const ids = ifcApi.GetLineIDsWithType(modelID, type);
  if (ids.size() === 0) {
    console.warn(`Missing: ${name}`);
    hierarchyValid = false;
  }
}

// Additionally verify the tree structure
const tree = await ifcApi.properties.getSpatialStructure(modelID);
if (!tree.children || tree.children.length === 0) {
  console.warn("Spatial tree has no children — possibly corrupt");
  hierarchyValid = false;
}
```

---

## AP-10: Assuming Consistent Property Set Names Across Models

**Wrong:**
```typescript
// Hardcoding property set names from one specific model
const psets = await ifcApi.properties.getPropertySets(modelID, wallID);
const wallCommon = psets.find(p => p.Name.value === "Pset_WallCommon");
const fireRating = wallCommon.HasProperties.find(
  p => p.Name.value === "FireRating"
);
```

**Why it fails:** Property set names and contents vary between authoring
tools (Revit, ArchiCAD, Tekla, etc.) and project standards. Some tools use
`Pset_WallCommon`, others use custom names. Properties within sets also
vary.

**Correct:**
```typescript
const psets = await ifcApi.properties.getPropertySets(modelID, wallID);
if (!psets) return;

// Search flexibly
for (const pset of psets) {
  const name = pset.Name?.value ?? "";
  if (/wall/i.test(name) || /common/i.test(name)) {
    // Found a wall-related property set
    const props = pset.HasProperties ?? [];
    for (const prop of props) {
      const propName = prop.Name?.value ?? "";
      const propValue = prop.NominalValue?.value ?? "N/A";
      console.log(`  ${propName}: ${propValue}`);
    }
  }
}
```
