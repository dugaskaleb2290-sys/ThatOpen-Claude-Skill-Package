# Model Analyzer — Analysis APIs Reference

## Classification APIs

### Classifier.byCategory(config?)

Groups loaded fragment items by IFC entity type.

```typescript
const classifier = components.get(OBC.Classifier);
await classifier.byCategory();
// Creates "Categories" classification with groups: "IFCWALL", "IFCSLAB", etc.
```

Config (`AddClassificationConfig`):
- `classificationName?: string` — Override default name "Categories"
- `modelIds?: RegExp[]` — Filter to specific models

### Classifier.byIfcBuildingStorey(config?)

Groups items by building storey via the ContainsElements relationship.

```typescript
await classifier.byIfcBuildingStorey();
// Creates "Storeys" classification with groups per storey name
```

### Classifier.byModel(config?)

Groups items by their parent FragmentsModel.

```typescript
await classifier.byModel();
// Creates "Models" classification with one group per loaded model
```

### Classifier.find(query)

Returns the intersection of specified classification groups as a ModelIdMap.

```typescript
const items = await classifier.find({
  Categories: ["IFCWALL", "IFCSLAB"],
  Storeys: ["Ground Floor"]
});
// Returns: ModelIdMap — walls AND slabs on the ground floor
```

Type: `ClassifierIntersectionInput` — keys are classification names, values
are arrays of group names.

### Classifier.list

```typescript
classifier.list: DataMap<string, DataMap<string, ClassificationGroupData>>
```

Access pattern:
```typescript
const categories = classifier.list.get("Categories");
if (categories) {
  for (const [groupName, groupData] of categories) {
    const items: ModelIdMap = await groupData.get();
    // groupName: "IFCWALL", "IFCSLAB", etc.
    // items: Map<string, Set<number>>
  }
}
```

---

## Data Extraction APIs

### FragmentsManager.getData(items, config?)

Extracts IFC property data for targeted elements via the web worker.

```typescript
const fragments = components.get(OBC.FragmentsManager);
const data = await fragments.getData(items);
// Returns: Record<string, ItemData[]>
// Keys: model UUID strings
// Values: arrays of ItemData objects
```

ALWAYS call `fragments.init(workerURL)` before using getData.

### web-ifc Properties Helper

High-level async methods on `ifcApi.properties`:

| Method | Signature | Returns |
|---|---|---|
| `getItemProperties` | `(modelID, expressID, recursive?, inverse?)` | All properties for one element |
| `getPropertySets` | `(modelID, expressID, recursive?)` | Property sets (Pset_) for one element |
| `getTypeProperties` | `(modelID, expressID, recursive?)` | Type object properties |
| `getMaterialsProperties` | `(modelID, expressID, recursive?)` | Material definitions |
| `getSpatialStructure` | `(modelID, includeProperties?)` | Full spatial hierarchy tree |

Spatial structure return format:
```typescript
{
  expressID: number,
  type: string,       // e.g., "IFCPROJECT"
  children: [
    {
      expressID: number,
      type: string,   // e.g., "IFCSITE"
      children: [...]
    }
  ]
}
```

---

## web-ifc Query APIs

### GetAllTypesOfModel(modelID)

Returns all IFC entity types present in the model.

```typescript
const types: Array<{ typeID: number, typeName: string }> =
  ifcApi.GetAllTypesOfModel(modelID);
```

Use for: generating a complete type inventory without querying each type.

### GetLineIDsWithType(modelID, type, includeInherited?)

Returns expressIDs of all entities of a given IFC type.

```typescript
import { IFCWALL } from "web-ifc";
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL);
// Returns: Vector<number> — use .size() and .get(i)
```

Set `includeInherited: true` to include subtypes (e.g., IFCWALL includes
IFCWALLSTANDARDCASE).

### GetLine(modelID, expressID, flatten?, inverse?)

Returns the full entity data for a single expressID.

```typescript
const wall = ifcApi.GetLine(modelID, expressID);
console.log(wall.Name?.value, wall.GlobalId?.value);
```

NEVER use `flatten: true` on large scopes — it recursively resolves all
references.

### GetRawLineData(modelID, expressID)

Returns unparsed entity data. Faster than GetLine when you only need type
and raw arguments.

```typescript
const raw = ifcApi.GetRawLineData(modelID, expressID);
// raw.ID, raw.type, raw.arguments
```

### GetModelSchema(modelID)

Returns the IFC schema version string.

```typescript
const schema = ifcApi.GetModelSchema(modelID);
// "IFC2X3" | "IFC4" | "IFC4X3"
```

### GetAllLines(modelID)

Returns all expressIDs in the model.

```typescript
const allIDs = ifcApi.GetAllLines(modelID);
console.log(`Total entities: ${allIDs.size()}`);
```

### GetHeaderLine(modelID, headerType)

Returns IFC file header information (file description, file name, etc.).

---

## ItemsFinder API

### Create Named Queries

```typescript
const finder = components.get(OBC.ItemsFinder);

const query = finder.create("Structural Walls", [
  {
    categories: [/WALL/],
    attributes: {
      queries: [{ name: /IsStructural/, value: /true/i }]
    }
  }
]);
```

### Execute Queries

```typescript
const items = await finder.getItems(
  [{ categories: [/WALL/, /SLAB/] }],
  { aggregation: "union" }
);
// Returns: ModelIdMap
```

### Auto-Generate Category Queries

```typescript
const categoryNames = await finder.addFromCategories();
// Returns: string[] of all geometric categories found
```

---

## GUID Utilities

For mapping between IFC GlobalId strings and expressIDs:

```typescript
// Build index first for performance
ifcApi.CreateIfcGuidToExpressIdMapping(modelID);

// Then convert
const expressID = ifcApi.GetExpressIdFromGuid(modelID, guid);
const guid = ifcApi.GetGuidFromExpressId(modelID, expressID);
```

ALWAYS call `CreateIfcGuidToExpressIdMapping` before batch GUID lookups.

---

## ModelIdMap Type

The universal interchange format for element sets:

```typescript
type ModelIdMap = Map<string, Set<number>>;
// Key: model UUID (string)
// Value: Set of local element IDs (expressIDs)
```

Counting elements in a ModelIdMap:
```typescript
function countItems(items: ModelIdMap): number {
  let total = 0;
  for (const ids of items.values()) {
    total += ids.size;
  }
  return total;
}
```
