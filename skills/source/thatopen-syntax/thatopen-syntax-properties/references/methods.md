# Property Queries and Classification API Reference

## Classifier

The Classifier component organizes fragment items into named
classification groups. Extends `Component`, implements `Disposable`.

**UUID:** `e25a7f3c-46c4-4a14-9d3d-5115f24ebeb7`

### Properties

| Property | Type | Description |
|---|---|---|
| `enabled` | `boolean` | Component activation state (default: `true`) |
| `list` | `DataMap<string, DataMap<string, ClassificationGroupData>>` | Nested map: classification name -> group name -> group data |
| `onDisposed` | `Event<unknown>` | Fires when the component is disposed |

### Classification Building Methods

#### `byCategory(config?: AddClassificationConfig): Promise<void>`

Creates groups for each IFC entity type found across loaded models.
Default classification name: `"Categories"`. Group names are IFC type
strings (e.g., `"IFCWALL"`, `"IFCSLAB"`, `"IFCDOOR"`).

Internally uses ItemsFinder to discover geometric categories and assigns
group queries for each.

#### `byIfcBuildingStorey(config?: AddClassificationConfig): Promise<void>`

Creates groups for each building storey. Default classification name:
`"Storeys"`. Group names are the storey `Name` attribute values
(e.g., `"Ground Floor"`, `"Level 1"`).

Internally uses `aggregateItemRelations()` with the `ContainsElements`
relationship to group elements by their containing storey.

#### `byModel(config?: AddClassificationConfig): Promise<void>`

Creates one group per loaded FragmentsModel. Default classification
name: `"Models"`. Group names are model identifiers.

Filters models using `modelIds` regex patterns from config if provided.

### AddClassificationConfig

```typescript
interface AddClassificationConfig {
  classificationName?: string;  // Override default classification name
  modelIds?: RegExp[];          // Only process models matching these patterns
}
```

### Query Methods

#### `find(data: ClassifierIntersectionInput): Promise<ModelIdMap>`

Returns the intersection of items across the specified classification
groups.

```typescript
type ClassifierIntersectionInput = {
  [classificationName: string]: string[];  // group names
};
```

When multiple classifications are specified, the result contains only
items that appear in ALL of the specified groups (set intersection).

#### `getGroupData(classification: string, group: string): ClassificationGroupData`

Retrieves or creates a group within a classification. Returns a
`ClassificationGroupData` object.

### ClassificationGroupData

Represents a single group within a classification.

| Member | Type | Description |
|---|---|---|
| `get()` | `Promise<ModelIdMap>` | Returns the items in this group |
| `query?` | `ClassificationGroupQuery` | Optional dynamic query config |

When a `query` is set, `get()` executes the query via ItemsFinder
to produce fresh results. Without a query, `get()` returns the
statically assigned items.

### ClassificationGroupQuery

```typescript
interface ClassificationGroupQuery {
  name: string;           // Name of an ItemsFinder query to execute
  config?: QueryTestConfig; // Optional query configuration
}
```

### Item Management Methods

#### `addGroupItems(classification: string, group: string, items: ModelIdMap): void`

Adds items to a specific group. Creates the group if it does not exist.

#### `removeItems(modelIdMap: ModelIdMap, config?: RemoveClassifierItemsConfig): void`

Removes items from the classifier. Without config, removes from ALL
classifications and ALL groups.

#### `setGroupQuery(classification: string, group: string, query: ClassificationGroupQuery): void`

Assigns a dynamic query to a group. The query references a named
ItemsFinder query by its `name` property.

### Aggregation Methods

#### `aggregateItems(classification: string, query: ItemsQueryParams, config?): Promise<void>`

Groups items matching a query into the specified classification. Uses an
aggregation callback (default: `defaultSaveFunction`) to determine which
group each item belongs to.

#### `aggregateItemRelations(classification: string, query: ItemsQueryParams, relation: string, config?: ClassifyItemRelationsConfig): Promise<void>`

Groups items by IFC relationships. Processes related items and places
them into groups based on the related entity's name.

### Utility Methods

#### `defaultSaveFunction(item: ItemData): null | string`

Default aggregation callback. Extracts the `value` from the item's
`Name` property. Returns `null` if the Name property is not available.

#### `dispose(): void`

Clears all classifications and releases resources.

---

## FragmentsManager.getData()

### Signature

```typescript
getData(
  items: ModelIdMap,
  config?: Partial<ItemsDataConfig>
): Promise<Record<string, ItemData[]>>
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `items` | `ModelIdMap` | Elements to query (model ID -> local ID sets) |
| `config` | `Partial<ItemsDataConfig>` | Optional: controls what data to retrieve |

### Return Value

`Record<string, ItemData[]>` — Keys are model UUID strings, values are
arrays of `ItemData` objects (one per queried element).

### ItemData

Interface from `@thatopen/fragments` representing the data of one item
in a fragments model. Contains the element's IFC properties organized
by their schema structure: attributes, property sets, quantity sets,
type information, and relationships.

### Performance Notes

- Runs in the web worker (non-blocking)
- Results are serialized back to the main thread via `postMessage`
- Large queries (thousands of items) can cause memory pressure during
  serialization — batch queries for large datasets

---

## ItemsFinder

Manages and executes queries to find items across loaded models.
Extends `Component`, implements `Disposable`, `Serializable`.

**UUID:** `0da7ad77-f734-42ca-942f-a074adfd1e3a`

### Properties

| Property | Type | Description |
|---|---|---|
| `enabled` | `boolean` | Component activation state (default: `true`) |
| `list` | `DataMap<string, FinderQuery>` | Named queries indexed by string keys |

### Methods

#### `create(name: string, queries: ItemsQueryParams[]): FinderQuery`

Creates and registers a named query with the specified parameters.
Returns a `FinderQuery` object that can be referenced by Classifier
via `setGroupQuery()`.

#### `getItems(queries: ItemsQueryParams[], config?): Promise<ModelIdMap>`

Executes queries and returns matching items.

Config options:

| Property | Type | Description |
|---|---|---|
| `modelIds` | `RegExp[]` | Filter: only search models matching these patterns |
| `items` | `ModelIdMap` | Restrict search to these items only |
| `aggregation` | `QueryResultAggregation` | `"union"` or `"intersection"` |

#### `addFromCategories(modelIds?: RegExp[]): Promise<string[]>`

Auto-generates queries for all geometric IFC categories found in loaded
models. Returns an array of category names created.

#### `export(): SerializedFinderQuery[]`

Serializes all named queries for persistence.

#### `import(result: SerializationResult): void`

Restores queries from serialized data.

### ItemsQueryParams

```typescript
interface ItemsQueryParams {
  categories?: RegExp[];      // Match IFC entity types (e.g., /WALL/, /SLAB/)
  relation?: {                // Filter by IFC relationship
    name: string;             // Relationship name
    query: ItemsQueryParams;  // Recursive: query the related items
  };
  attributes?: {              // Filter by element attributes
    queries: Array<{
      name: RegExp;           // Attribute name pattern
      value: RegExp;          // Attribute value pattern
    }>;
  };
}
```

**Supported relationship names:**
- `"ContainedInStructure"` — IfcRelContainedInSpatialStructure
- `"ContainsElements"` — Inverse of ContainedInStructure
- `"Aggregates"` — IfcRelAggregates
- `"IsDecomposedBy"` — Inverse of Aggregates

---

## ModelIdMap (recap)

```typescript
type ModelIdMap = Record<string, Set<number>>;
```

The universal data structure for targeting items. Keys are model UUID
strings, values are sets of local element IDs. Used as input for
`getData()`, `highlight()`, `Hider.set()`, and as output from
`Classifier.find()` and `ItemsFinder.getItems()`.
