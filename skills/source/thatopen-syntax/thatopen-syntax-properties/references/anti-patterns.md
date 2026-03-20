# Property Queries and Classification Anti-Patterns

## 1. Classifying Before Loading

**WRONG — calling classification methods with no loaded models:**

```typescript
const classifier = components.get(OBC.Classifier);

// No models loaded yet!
await classifier.byCategory();
await classifier.byIfcBuildingStorey();

// Result: empty classifications, no groups created
const items = await classifier.find({ Categories: ["IFCWALL"] });
// items is empty — no walls found because no models were loaded
```

**CORRECT — classify after model loading completes:**

```typescript
const classifier = components.get(OBC.Classifier);
const ifcLoader = components.get(OBC.IfcLoader);

// Load model first
await ifcLoader.setup();
const model = await ifcLoader.load(data, true, "Building");

// NOW classify
await classifier.byCategory();
await classifier.byIfcBuildingStorey();
```

**Rule:** ALWAYS load at least one model before calling classification
methods. Classification reads data from loaded FragmentsModels.

---

## 2. Not Re-Classifying After New Model Loads

**WRONG — assuming old classifications include new models:**

```typescript
// Load model A and classify
const modelA = await ifcLoader.load(dataA, true, "Arch");
await classifier.byCategory();

// Load model B later
const modelB = await ifcLoader.load(dataB, true, "Struct");

// BUG: classifier still only has categories from model A
const allWalls = await classifier.find({ Categories: ["IFCWALL"] });
// Missing walls from model B!
```

**CORRECT — re-classify after each new model load:**

```typescript
const modelA = await ifcLoader.load(dataA, true, "Arch");
await classifier.byCategory();

const modelB = await ifcLoader.load(dataB, true, "Struct");
await classifier.byCategory();  // Re-run to include model B
```

**Rule:** ALWAYS re-run classification methods after loading additional
models. Classifications are NOT automatically updated when new models
are added.

---

## 3. Manually Iterating classifier.list Instead of Using find()

**WRONG — building ModelIdMap by hand from classification groups:**

```typescript
const classifier = components.get(OBC.Classifier);

// Manually digging into the data structure
const categories = classifier.list.get("Categories");
const storeys = classifier.list.get("Storeys");
const wallGroup = categories?.get("IFCWALL");
const storeyGroup = storeys?.get("Ground Floor");

// Manually intersecting... error-prone and verbose
const wallItems = wallGroup ? await wallGroup.get() : {};
const storeyItems = storeyGroup ? await storeyGroup.get() : {};
// Now what? Manual intersection logic? Bug city.
```

**CORRECT — use find() for intersection queries:**

```typescript
const items = await classifier.find({
  Categories: ["IFCWALL"],
  Storeys: ["Ground Floor"]
});
// find() handles intersection correctly
```

**Rule:** ALWAYS use `classifier.find()` for cross-classification
queries. NEVER manually iterate `classifier.list` to build intersections.

---

## 4. Querying getData Without Worker Initialization

**WRONG — calling getData before FragmentsManager.init():**

```typescript
const fragments = components.get(OBC.FragmentsManager);
// MISSING: fragments.init(workerURL)

const items = { [model.modelId]: new Set([42]) };
const data = await fragments.getData(items);
// Result: crash, silent failure, or empty result
```

**CORRECT:**

```typescript
const fragments = components.get(OBC.FragmentsManager);
fragments.init("https://unpkg.com/@thatopen/fragments@3.3.6/dist/Worker/worker.mjs");

// Worker initialized — safe to query
const data = await fragments.getData(items);
```

**Rule:** ALWAYS initialize the FragmentsManager worker before calling
`getData()`. The property extraction runs entirely in the worker thread.

---

## 5. Querying Too Many Items at Once

**WRONG — extracting properties for entire model without filtering:**

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Build a ModelIdMap with ALL elements in ALL models
const allItems: OBC.ModelIdMap = {};
for (const [modelId, model] of fragments.list) {
  // Collecting thousands of IDs...
  allItems[modelId] = new Set(/* all local IDs */);
}

// Querying properties for 50,000 elements at once
const data = await fragments.getData(allItems);
// Result: massive serialization overhead, UI freezes, memory spike
```

**CORRECT — classify first, then query targeted subsets:**

```typescript
const classifier = components.get(OBC.Classifier);
await classifier.byCategory();

// Query only the elements you need
const walls = await classifier.find({ Categories: ["IFCWALL"] });
const wallData = await fragments.getData(walls);
```

**Rule:** ALWAYS use Classifier or ItemsFinder to narrow down items
before calling `getData()`. NEVER extract properties for all elements
in a large model at once.

---

## 6. Using Wrong Classification or Group Names

**WRONG — typo in classification or group name:**

```typescript
await classifier.byCategory();
await classifier.byIfcBuildingStorey();

// Typo: "Category" instead of "Categories"
const items = await classifier.find({
  Category: ["IFCWALL"]  // WRONG: should be "Categories"
});
// Result: empty ModelIdMap — no classification named "Category" exists

// Wrong case in group name
const items2 = await classifier.find({
  Categories: ["IfcWall"]  // WRONG: should be "IFCWALL" (uppercase)
});
```

**CORRECT — use exact default names:**

```typescript
// Default classification names (case-sensitive):
// byCategory()           -> "Categories"
// byIfcBuildingStorey()  -> "Storeys"
// byModel()              -> "Models"

const items = await classifier.find({
  Categories: ["IFCWALL"],    // Uppercase IFC type names
  Storeys: ["Ground Floor"]   // Exact storey name from the IFC model
});
```

**Rule:** ALWAYS use the exact default classification names:
`"Categories"`, `"Storeys"`, `"Models"`. Group names for categories
are uppercase IFC type strings (e.g., `"IFCWALL"`, not `"IfcWall"`).
Storey group names match the `Name` attribute in the IFC model exactly.

---

## 7. Forgetting to Await Async Classification Methods

**WRONG — not awaiting classification before querying:**

```typescript
// byCategory is async! Missing await
classifier.byCategory();
classifier.byIfcBuildingStorey();

// Classification not yet complete — find() sees empty groups
const items = await classifier.find({
  Categories: ["IFCWALL"]
});
// Result: empty or incomplete ModelIdMap
```

**CORRECT:**

```typescript
await classifier.byCategory();
await classifier.byIfcBuildingStorey();

const items = await classifier.find({
  Categories: ["IFCWALL"]
});
```

**Rule:** ALWAYS await `byCategory()`, `byIfcBuildingStorey()`,
`byModel()`, and `find()`. All classification operations are async.

---

## 8. Using ItemsFinder Regex Without Understanding Matching

**WRONG — overly broad regex matches wrong categories:**

```typescript
const finder = components.get(OBC.ItemsFinder);

// /WALL/ also matches IFCWALLSTANDARDCASE, IFCCURTAINWALL, etc.
const items = await finder.getItems([
  { categories: [/WALL/] }
]);
// May include curtain walls when you only wanted standard walls
```

**CORRECT — use precise regex when specificity matters:**

```typescript
// Match only IFCWALL (not subtypes)
const items = await finder.getItems([
  { categories: [/^IFCWALL$/] }
]);

// Or include standard case explicitly
const items2 = await finder.getItems([
  { categories: [/^IFCWALL$/, /^IFCWALLSTANDARDCASE$/] }
]);
```

**Rule:** ALWAYS consider regex matching scope. Use anchors (`^`, `$`)
when you need exact type matching. Unanchored patterns like `/WALL/`
match any type containing "WALL".

---

## 9. Not Disposing Classifier on Model Removal

**WRONG — disposing a model without cleaning classifier data:**

```typescript
const classifier = components.get(OBC.Classifier);
await classifier.byCategory();

// Dispose a model
fragments.disposeModel(model.modelId);

// BUG: classifier still references items from the disposed model
const items = await classifier.find({ Categories: ["IFCWALL"] });
// items may contain IDs from the disposed model — stale references
```

**CORRECT — remove items or re-classify after model disposal:**

```typescript
// Option A: Remove model's items from classifier
const modelItems: OBC.ModelIdMap = {
  [model.modelId]: new Set(/* all IDs */)
};
classifier.removeItems(modelItems);
fragments.disposeModel(model.modelId);

// Option B: Re-classify from scratch after disposal
fragments.disposeModel(model.modelId);
await classifier.byCategory();
await classifier.byIfcBuildingStorey();
```

**Rule:** ALWAYS clean up classifier data when disposing models. Either
call `removeItems()` before disposal or re-run classification methods
after disposal.

---

## Summary Table

| Anti-Pattern | Consequence | Rule |
|---|---|---|
| Classify before loading | Empty classifications | ALWAYS load models first |
| Skip re-classification | Missing new model data | ALWAYS re-classify after new loads |
| Manual list iteration | Incorrect intersections | ALWAYS use find() for queries |
| getData without worker init | Silent failure or crash | ALWAYS init worker first |
| Query all items at once | Memory spike, UI freeze | ALWAYS filter items first |
| Wrong classification names | Empty results | ALWAYS use exact default names |
| Missing await on async methods | Incomplete results | ALWAYS await classification calls |
| Overly broad regex | Wrong type matches | ALWAYS use anchors for exact matching |
| Stale classifier after disposal | Stale references | ALWAYS clean up after model disposal |
