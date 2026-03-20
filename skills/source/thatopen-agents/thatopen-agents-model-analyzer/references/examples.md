# Model Analyzer — Complete Examples

## Example 1: Full Model Summary

Produces a complete overview of a loaded model.

```typescript
import * as OBC from "@thatopen/components";
import {
  IFCWALL, IFCSLAB, IFCDOOR, IFCWINDOW, IFCBEAM, IFCCOLUMN,
  IFCROOF, IFCSTAIR, IFCFURNISHINGELEMENT, IFCSPACE,
  IFCPROJECT, IFCSITE, IFCBUILDING, IFCBUILDINGSTOREY,
} from "web-ifc";

const fragments = components.get(OBC.FragmentsManager);
const classifier = components.get(OBC.Classifier);
const loader = components.get(OBC.IfcLoader);
const ifcApi = loader.webIfc;

// Assume model is already loaded and we have the web-ifc modelID
// Step 1: Schema and type overview
const schema = ifcApi.GetModelSchema(modelID);
const allTypes = ifcApi.GetAllTypesOfModel(modelID);
const totalEntities = ifcApi.GetAllLines(modelID).size();

console.log(`Schema: ${schema}`);
console.log(`Entity types present: ${allTypes.length}`);
console.log(`Total entities: ${totalEntities}`);

// Step 2: Element counts by type
const buildingElements = [
  { type: IFCWALL, name: "IFCWALL" },
  { type: IFCSLAB, name: "IFCSLAB" },
  { type: IFCDOOR, name: "IFCDOOR" },
  { type: IFCWINDOW, name: "IFCWINDOW" },
  { type: IFCBEAM, name: "IFCBEAM" },
  { type: IFCCOLUMN, name: "IFCCOLUMN" },
  { type: IFCROOF, name: "IFCROOF" },
  { type: IFCSTAIR, name: "IFCSTAIR" },
  { type: IFCFURNISHINGELEMENT, name: "IFCFURNISHINGELEMENT" },
  { type: IFCSPACE, name: "IFCSPACE" },
];

const inventory: Record<string, number> = {};
let totalElements = 0;

for (const { type, name } of buildingElements) {
  const ids = ifcApi.GetLineIDsWithType(modelID, type);
  const count = ids.size();
  if (count > 0) {
    inventory[name] = count;
    totalElements += count;
  }
}

// Step 3: Spatial structure
const spatialTree = await ifcApi.properties.getSpatialStructure(modelID);

function formatTree(node: any, indent = 0): string {
  const prefix = "  ".repeat(indent);
  let result = `${prefix}${node.type} [#${node.expressID}]\n`;
  if (node.children) {
    for (const child of node.children) {
      result += formatTree(child, indent + 1);
    }
  }
  return result;
}

console.log("Spatial Structure:");
console.log(formatTree(spatialTree));

// Step 4: Storey breakdown
await classifier.byCategory();
await classifier.byIfcBuildingStorey();

const storeys = classifier.list.get("Storeys");
if (storeys) {
  console.log("\nElements per storey:");
  for (const [storeyName, groupData] of storeys) {
    const items = await groupData.get();
    let count = 0;
    for (const ids of Object.values(items)) {
      count += (ids as Set<number>).size;
    }
    console.log(`  ${storeyName}: ${count} elements`);
  }
}
```

---

## Example 2: Property Report for Walls

Generates a property set report for all wall elements.

```typescript
import * as OBC from "@thatopen/components";
import { IFCWALL } from "web-ifc";

const fragments = components.get(OBC.FragmentsManager);
const classifier = components.get(OBC.Classifier);
const loader = components.get(OBC.IfcLoader);
const ifcApi = loader.webIfc;

// Classify first
await classifier.byCategory();

// Get walls via classifier
const wallItems = await classifier.find({ Categories: ["IFCWALL"] });

// Extract properties via fragments (high-level API)
const wallData = await fragments.getData(wallItems);

for (const [modelId, itemDataArray] of Object.entries(wallData)) {
  console.log(`Model: ${modelId}`);
  for (const itemData of itemDataArray) {
    console.log("  Element data:", JSON.stringify(itemData, null, 2));
  }
}

// Alternatively: detailed property sets via web-ifc (low-level API)
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
const psetReport: Record<string, number> = {};
const BATCH_SIZE = 50;

for (let batch = 0; batch < wallIDs.size(); batch += BATCH_SIZE) {
  const end = Math.min(batch + BATCH_SIZE, wallIDs.size());
  for (let i = batch; i < end; i++) {
    const expressID = wallIDs.get(i);
    const psets = await ifcApi.properties.getPropertySets(
      modelID, expressID, false
    );
    if (psets) {
      for (const pset of psets) {
        const name = pset.Name?.value || "Unnamed";
        psetReport[name] = (psetReport[name] || 0) + 1;
      }
    }
  }
}

console.log("\nProperty Set Occurrence Report:");
for (const [name, count] of Object.entries(psetReport).sort(
  (a, b) => b[1] - a[1]
)) {
  console.log(`  ${name}: ${count} occurrences`);
}
```

---

## Example 3: Element Inventory by Storey

Produces a matrix of element types vs building storeys.

```typescript
import * as OBC from "@thatopen/components";

const classifier = components.get(OBC.Classifier);

// Classify
await classifier.byCategory();
await classifier.byIfcBuildingStorey();

const categories = classifier.list.get("Categories");
const storeys = classifier.list.get("Storeys");

if (!categories || !storeys) {
  console.error("Classification data not available");
  throw new Error("Run classifier before inventory");
}

// Build the matrix
const matrix: Record<string, Record<string, number>> = {};
const categoryNames = Array.from(categories.keys());
const storeyNames = Array.from(storeys.keys());

for (const storeyName of storeyNames) {
  matrix[storeyName] = {};
  for (const categoryName of categoryNames) {
    // Cross-query: elements of this type on this storey
    const items = await classifier.find({
      Categories: [categoryName],
      Storeys: [storeyName],
    });
    let count = 0;
    for (const ids of Object.values(items)) {
      count += (ids as Set<number>).size;
    }
    if (count > 0) {
      matrix[storeyName][categoryName] = count;
    }
  }
}

// Output as table
console.log("\nElement Inventory by Storey:");
console.log("Storey | " + categoryNames.join(" | "));
console.log("-".repeat(70));
for (const storeyName of storeyNames) {
  const row = categoryNames.map(
    (cat) => String(matrix[storeyName][cat] || 0).padStart(5)
  );
  console.log(`${storeyName.padEnd(20)} | ${row.join(" | ")}`);
}
```

---

## Example 4: Model Validation Check

Runs quality checks and outputs a validation summary.

```typescript
import * as OBC from "@thatopen/components";
import {
  IFCWALL, IFCSLAB, IFCDOOR, IFCWINDOW,
  IFCPROJECT, IFCSITE, IFCBUILDING, IFCBUILDINGSTOREY,
} from "web-ifc";

const classifier = components.get(OBC.Classifier);
const loader = components.get(OBC.IfcLoader);
const ifcApi = loader.webIfc;

await classifier.byCategory();
await classifier.byIfcBuildingStorey();

const results: Array<{ check: string; status: string; detail: string }> = [];

// Check 1: Required spatial elements
const spatialChecks = [
  { type: IFCPROJECT, name: "IfcProject" },
  { type: IFCSITE, name: "IfcSite" },
  { type: IFCBUILDING, name: "IfcBuilding" },
  { type: IFCBUILDINGSTOREY, name: "IfcBuildingStorey" },
];

for (const { type, name } of spatialChecks) {
  const ids = ifcApi.GetLineIDsWithType(modelID, type);
  const count = ids.size();
  results.push({
    check: `${name} present`,
    status: count > 0 ? "PASS" : "WARN",
    detail: `Found ${count} instance(s)`,
  });
}

// Check 2: Elements with property sets (sample first 20 walls)
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
let wallsWithPsets = 0;
const sampleSize = Math.min(wallIDs.size(), 20);

for (let i = 0; i < sampleSize; i++) {
  const psets = await ifcApi.properties.getPropertySets(
    modelID, wallIDs.get(i), false
  );
  if (psets && psets.length > 0) wallsWithPsets++;
}

const psetPercent = sampleSize > 0
  ? Math.round((wallsWithPsets / sampleSize) * 100)
  : 0;
results.push({
  check: "Walls with property sets",
  status: psetPercent > 80 ? "PASS" : "WARN",
  detail: `${wallsWithPsets}/${sampleSize} sampled (${psetPercent}%)`,
});

// Check 3: Storey assignment
const storeys = classifier.list.get("Storeys");
const storeyCount = storeys?.size ?? 0;
results.push({
  check: "Storey classification",
  status: storeyCount > 0 ? "PASS" : "WARN",
  detail: `${storeyCount} storey(s) detected`,
});

// Output validation report
console.log("\n=== VALIDATION RESULTS ===");
for (const r of results) {
  console.log(`[${r.status}] ${r.check}: ${r.detail}`);
}
```

---

## Example 5: Export Analysis Data as JSON

Structures analysis results for downstream use.

```typescript
import * as OBC from "@thatopen/components";

const fragments = components.get(OBC.FragmentsManager);
const classifier = components.get(OBC.Classifier);
const loader = components.get(OBC.IfcLoader);
const ifcApi = loader.webIfc;

await classifier.byCategory();
await classifier.byIfcBuildingStorey();

// Build exportable report
const report = {
  schema: ifcApi.GetModelSchema(modelID),
  totalTypes: ifcApi.GetAllTypesOfModel(modelID).length,
  totalEntities: ifcApi.GetAllLines(modelID).size(),
  categories: {} as Record<string, number>,
  storeys: {} as Record<string, number>,
  generatedAt: new Date().toISOString(),
};

// Populate categories
const categories = classifier.list.get("Categories");
if (categories) {
  for (const [name, groupData] of categories) {
    const items = await groupData.get();
    let count = 0;
    for (const ids of Object.values(items)) {
      count += (ids as Set<number>).size;
    }
    report.categories[name] = count;
  }
}

// Populate storeys
const storeys = classifier.list.get("Storeys");
if (storeys) {
  for (const [name, groupData] of storeys) {
    const items = await groupData.get();
    let count = 0;
    for (const ids of Object.values(items)) {
      count += (ids as Set<number>).size;
    }
    report.storeys[name] = count;
  }
}

// Output as JSON
const jsonReport = JSON.stringify(report, null, 2);
console.log(jsonReport);
```
