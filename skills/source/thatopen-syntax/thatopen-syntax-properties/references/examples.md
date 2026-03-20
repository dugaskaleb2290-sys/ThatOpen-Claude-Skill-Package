# Property Queries and Classification Examples

## 1. Classify by Category and Storey

ALWAYS classify after models are loaded.

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
const fragments = components.get(OBC.FragmentsManager);
const classifier = components.get(OBC.Classifier);

// Assume model is already loaded (see thatopen-syntax-ifc-loading)
// ...

// Classify all loaded models by IFC entity type
await classifier.byCategory();

// Classify by building storey
await classifier.byIfcBuildingStorey();

// Classify by source model
await classifier.byModel();

// Now classifier.list has three classifications:
// "Categories" -> {"IFCWALL": ..., "IFCSLAB": ..., "IFCDOOR": ...}
// "Storeys"    -> {"Ground Floor": ..., "Level 1": ..., "Level 2": ...}
// "Models"     -> {"model-uuid-1": ..., "model-uuid-2": ...}
```

---

## 2. Find Items with Classifier.find()

Use `find()` to get a ModelIdMap from classification intersections.

```typescript
const classifier = components.get(OBC.Classifier);

// Find all walls
const allWalls = await classifier.find({
  Categories: ["IFCWALL"]
});

// Find all items on the ground floor
const groundFloorItems = await classifier.find({
  Storeys: ["Ground Floor"]
});

// Find walls on the ground floor (intersection)
const groundFloorWalls = await classifier.find({
  Categories: ["IFCWALL"],
  Storeys: ["Ground Floor"]
});

// Find walls AND slabs on a specific storey
const structuralItems = await classifier.find({
  Categories: ["IFCWALL", "IFCSLAB"],
  Storeys: ["Level 1"]
});
```

---

## 3. Extract Properties with getData()

```typescript
const fragments = components.get(OBC.FragmentsManager);
const classifier = components.get(OBC.Classifier);

// First: classify
await classifier.byCategory();
await classifier.byIfcBuildingStorey();

// Second: find items of interest
const wallItems = await classifier.find({
  Categories: ["IFCWALL"],
  Storeys: ["Ground Floor"]
});

// Third: extract IFC property data
const wallData = await fragments.getData(wallItems);

// Process results
for (const [modelId, itemDataArray] of Object.entries(wallData)) {
  console.log(`Model ${modelId}: ${itemDataArray.length} walls`);
  for (const itemData of itemDataArray) {
    console.log("Wall properties:", itemData);
  }
}
```

---

## 4. getData for Specific Elements

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Target specific elements by local ID
const items: OBC.ModelIdMap = {
  [model.modelId]: new Set([42, 43, 44])
};

const data = await fragments.getData(items);

for (const [modelId, elements] of Object.entries(data)) {
  for (const element of elements) {
    console.log(element);
  }
}
```

---

## 5. ItemsFinder — Search by Category and Relationship

```typescript
const finder = components.get(OBC.ItemsFinder);

// Simple category search: find all walls
const walls = await finder.getItems([
  { categories: [/WALL/] }
]);

// Multi-category search: walls and slabs
const structural = await finder.getItems([
  { categories: [/WALL/, /SLAB/] }
]);

// Filter by model ID pattern
const archWalls = await finder.getItems(
  [{ categories: [/WALL/] }],
  { modelIds: [/arch/i] }
);
```

---

## 6. ItemsFinder — Relationship Queries

Find items by their IFC relationships using nested queries.

```typescript
const finder = components.get(OBC.ItemsFinder);

// Find walls contained in a specific storey
const firstFloorWalls = await finder.getItems([
  {
    categories: [/WALL/],
    relation: {
      name: "ContainedInStructure",
      query: {
        categories: [/STOREY/],
        attributes: {
          queries: [{ name: /Name/, value: /First Floor/ }]
        }
      }
    }
  }
]);

// Find all elements in spaces named "Office"
const officeElements = await finder.getItems([
  {
    relation: {
      name: "ContainedInStructure",
      query: {
        categories: [/SPACE/],
        attributes: {
          queries: [{ name: /Name/, value: /Office/ }]
        }
      }
    }
  }
]);
```

---

## 7. ItemsFinder — Named Queries for Reuse

```typescript
const finder = components.get(OBC.ItemsFinder);

// Create a reusable named query
finder.create("Ground Floor Walls", [
  {
    categories: [/WALL/],
    relation: {
      name: "ContainedInStructure",
      query: {
        categories: [/STOREY/],
        attributes: {
          queries: [{ name: /Name/, value: /Ground|00|GF/i }]
        }
      }
    }
  }
]);

// Execute by name later (through Classifier integration)
const classifier = components.get(OBC.Classifier);
classifier.setGroupQuery("Custom", "Ground Floor Walls", {
  name: "Ground Floor Walls"
});

// Now find() will execute the query dynamically
const items = await classifier.find({ Custom: ["Ground Floor Walls"] });
```

---

## 8. Auto-Generate Category Queries

```typescript
const finder = components.get(OBC.ItemsFinder);

// Create queries for all IFC categories present in loaded models
const categories = await finder.addFromCategories();
console.log("Found categories:", categories);
// ["IFCWALL", "IFCSLAB", "IFCDOOR", "IFCWINDOW", ...]

// Now use them
const slabs = await finder.getItems([{ categories: [/SLAB/] }]);
```

---

## 9. Custom Classification Groups

Build custom classifications for domain-specific grouping.

```typescript
const classifier = components.get(OBC.Classifier);
const fragments = components.get(OBC.FragmentsManager);

// Classify by category first
await classifier.byCategory();

// Get all walls
const wallItems = await classifier.find({ Categories: ["IFCWALL"] });

// Get property data for walls
const wallData = await fragments.getData(wallItems);

// Build custom groups based on property values
for (const [modelId, elements] of Object.entries(wallData)) {
  for (const element of elements) {
    // Example: group by some property value
    // (actual structure depends on the IFC model)
    const groupName = "External Walls"; // derive from property data
    const itemMap: OBC.ModelIdMap = {
      [modelId]: new Set([/* element local IDs */])
    };
    classifier.addGroupItems("Fire Rating", groupName, itemMap);
  }
}

// Now query custom groups
const ratedWalls = await classifier.find({
  "Fire Rating": ["External Walls"]
});
```

---

## 10. Iterate Classification Groups

```typescript
const classifier = components.get(OBC.Classifier);

await classifier.byCategory();
await classifier.byIfcBuildingStorey();

// Iterate all classifications and groups
for (const [classificationName, groups] of classifier.list) {
  console.log(`Classification: ${classificationName}`);
  for (const [groupName, groupData] of groups) {
    const items = await groupData.get();
    const totalItems = Object.values(items)
      .reduce((sum, set) => sum + set.size, 0);
    console.log(`  ${groupName}: ${totalItems} items`);
  }
}
```

---

## 11. Spatial Structure Traversal

Access the building's spatial hierarchy through storey classification.

```typescript
const classifier = components.get(OBC.Classifier);
const fragments = components.get(OBC.FragmentsManager);

await classifier.byIfcBuildingStorey();

const storeys = classifier.list.get("Storeys");
if (storeys) {
  for (const [storeyName, groupData] of storeys) {
    const items = await groupData.get();

    // Count elements per storey
    let count = 0;
    for (const set of Object.values(items)) {
      count += set.size;
    }

    // Get property data for all elements on this storey
    const data = await fragments.getData(items);

    console.log(`${storeyName}: ${count} elements`);
  }
}
```

---

## 12. Serialize and Restore Queries

```typescript
const finder = components.get(OBC.ItemsFinder);

// Create queries
finder.create("Structural Walls", [
  { categories: [/WALL/], attributes: {
    queries: [{ name: /LoadBearing/, value: /true/i }]
  }}
]);

// Export for storage (e.g., localStorage, server)
const serialized = finder.export();
localStorage.setItem("queries", JSON.stringify(serialized));

// Restore in a new session
const saved = JSON.parse(localStorage.getItem("queries")!);
await finder.import(saved);
```
