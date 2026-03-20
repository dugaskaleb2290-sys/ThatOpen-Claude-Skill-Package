# BCF & Viewpoints — Method Reference

## BCFTopics

**UUID**: `de977976-e4f6-4e4f-a01a-204727839802`
**Package**: `@thatopen/components`
**Implements**: `Component`, `Disposable`, `Configurable<BCFTopicsConfigManager, BCFTopicsConfig>`

### Constructor

```typescript
const bcfTopics = components.get(OBC.BCFTopics);
```

ALWAYS use `components.get()`. NEVER instantiate directly.

### setup(config?)

```typescript
setup(config?: Partial<BCFTopicsConfig>): void
```

Initializes the component with configuration. Fires `onSetup` event.
ALWAYS call before `create()`, `load()`, or `export()`.

### create(data?)

```typescript
create(data?: Partial<BCFTopic>): Topic
```

Creates a new Topic instance. Auto-generates GUID, sets `creationDate`
to now and `creationAuthor` from config. Optional `data` initializes
properties via `topic.set(data)`.

Returns the created `Topic` instance, which is also added to `list`.

### load(data)

```typescript
async load(data: Uint8Array): Promise<{ viewpoints: Viewpoint[]; topics: Topic[] }>
```

Imports a BCF zip file. Parses version from `bcf.version` XML inside
the zip. Processes markup files, viewpoints, and document references.
Fires `onBCFImported` with the imported topics.

Returns both the imported topics and their associated viewpoints.

### export(topics?)

```typescript
async export(topics?: Iterable<Topic>): Promise<Blob>
```

Exports topics to a BCF zip file (Blob). If `topics` is omitted,
exports all topics in `list`. The zip contains:
- `bcf.version` (XML with version number)
- `bcf.extensions` (XML with allowed types/statuses/priorities/etc.)
- Per topic: `{guid}/markup.bcf`, viewpoint XML files, snapshot images

### updateExtensions()

```typescript
updateExtensions(): void
```

Synchronizes config sets (types, statuses, priorities, labels, stages,
users) with values currently in use across all topics. Call after
programmatic topic changes to keep config current.

### updateViewpointReferences()

```typescript
updateViewpointReferences(): void
```

Removes stale viewpoint GUIDs from topics when the referenced
viewpoints no longer exist in the Viewpoints component list.

### dispose()

```typescript
dispose(): void
```

Clears `list` and `documents`, fires `onDisposed`.

### Computed Getters

```typescript
get usedTypes(): Set<string>        // All topic.type values
get usedStatuses(): Set<string>     // All topic.status values
get usedPriorities(): Set<string>   // All non-null topic.priority values
get usedStages(): Set<string>       // All non-null topic.stage values
get usedUsers(): Set<string>        // All authors + comment authors
get usedLabels(): Set<string>       // All labels across topics
```

---

## BCFTopicsConfig

```typescript
interface BCFTopicsConfig {
  version: "2.1" | "3";
  author: string;
  types: Set<string>;
  statuses: Set<string>;
  priorities: Set<string>;
  labels: Set<string>;
  stages: Set<string>;
  users: Set<string>;
  includeSelectionTag: boolean;
  updateExtensionsOnImport: boolean;
  strict: boolean;
  includeAllExtensionsOnExport: boolean;
  fallbackVersionOnImport: BCFVersion | null;
  ignoreIncompleteTopicsOnImport: boolean;
  exportCustomDataAsLabels: boolean;
}
```

---

## Topic

### Constructor

Topics are created via `bcfTopics.create()`, NEVER directly.

### set(data)

```typescript
set(data: Partial<BCFTopic>): Topic
```

Bulk-update topic properties. Skips `guid` to prevent identity changes.
Triggers reactive events (unlike direct property assignment).

### createComment(text, viewpoint?)

```typescript
createComment(text: string, viewpoint?: string): Comment
```

Creates a Comment associated with this topic. `author` and `date` are
auto-set from config. Optional `viewpoint` is a viewpoint GUID.

### toJSON()

```typescript
toJSON(): BCFApiTopic
```

Serializes to BCF API format (JSON). Converts dates to ISO strings,
strips undefined fields.

### serialize()

```typescript
serialize(): string
```

Generates BCF markup XML for this topic. Format depends on the
BCFTopics config version ("2.1" or "3").

---

## BCFTopic Type

```typescript
interface BCFTopic {
  guid: string;
  serverAssignedId?: string;
  type: string;
  status: string;
  title: string;
  priority?: string;
  index?: number;
  labels: Set<string>;
  creationDate: Date;
  creationAuthor: string;
  modifiedDate?: Date;
  modifiedAuthor?: string;
  dueDate?: Date;
  assignedTo?: string;
  description?: string;
  stage?: string;
}
```

---

## Comment

### Properties

| Property | Type | Mutable | Auto-updated |
|---|---|---|---|
| `guid` | `string` | No | — |
| `date` | `Date` | No | — |
| `author` | `string` | No | — |
| `comment` | `string` | Yes (setter) | `modifiedDate`, `modifiedAuthor` |
| `viewpoint` | `string?` | Yes | — |
| `modifiedDate` | `Date?` | — | On comment change |
| `modifiedAuthor` | `string?` | — | On comment change |

### toJSON()

```typescript
toJSON(): BCFApiComment
```

Serializes to JSON with ISO date strings.

---

## Document References

```typescript
interface DocumentReference {
  type: "internal" | "external";
  description?: string;
}

interface InternalDocumentReference extends DocumentReference {
  type: "internal";
  fileName: string;
  data: Uint8Array;
}

interface ExternalDocumentReference extends DocumentReference {
  type: "external";
  url: string;
}
```

Stored in `bcfTopics.documents` DataMap, indexed by auto-generated GUID.

---

## Viewpoints

**UUID**: `ee867824-a796-408d-8aa0-4e5962a83c66`
**Package**: `@thatopen/components`
**Implements**: `Component`, `Disposable`, `Configurable`

### Constructor

```typescript
const viewpoints = components.get(OBC.Viewpoints);
viewpoints.world = world; // REQUIRED
```

### create(data?)

```typescript
create(data?: Partial<BCFViewpoint>): Viewpoint
```

Creates a new Viewpoint instance. Optional `data` populates from
BCF viewpoint data (used during import).

### getSnapshotExtension(name)

```typescript
getSnapshotExtension(name: string): string
```

Examines snapshot header bytes to detect format. Returns `"png"` or
`"jpeg"` (default).

### dispose()

```typescript
dispose(): void
```

Clears `list` and `snapshots`, fires `onDisposed`.

---

## Viewpoint Instance

### updateCamera(takeSnapshot?)

```typescript
updateCamera(takeSnapshot?: boolean): void
```

Captures the current world camera position, direction, projection, and
field of view (or scale for ortho). Optionally takes a snapshot.

### go(config?)

```typescript
async go(config?: object): Promise<void>
```

Applies the viewpoint to the world: sets camera position/direction,
applies visibility, colorization, and clipping state.

### takeSnapshot()

```typescript
takeSnapshot(): void
```

Captures the current world renderer canvas as a Uint8Array and stores
it in the Viewpoints manager `snapshots` map keyed by this viewpoint's
GUID.

### applyVisibility()

```typescript
applyVisibility(): void
```

Enforces `defaultVisibility`, exception components, and selection
components using the Hider component.

### setColorizationState(state)

```typescript
setColorizationState(state: boolean): void
```

When `true`, applies `componentColors` highlighting via FragmentsManager.
When `false`, resets colorization.

### updateClippingPlanes()

```typescript
updateClippingPlanes(): void
```

Syncs the viewpoint's `clippingPlanes` from the current Clipper state.

### toJSON()

```typescript
toJSON(): BCFViewpoint
```

Serializes to BCFViewpoint data object.

### serialize(version)

```typescript
serialize(version: "2.1" | "3"): string
```

Generates BCF viewpoint XML. Coordinate transformations differ between
v2.1 and v3.0.

---

## BCFViewpoint Type

```typescript
interface BCFViewpoint {
  perspective_camera?: ViewpointPerspectiveCamera;
  orthogonal_camera?: ViewpointOrthogonalCamera;
  components?: ViewpointComponents;
  snapshot?: ViewpointSnapshot;
  lines?: ViewpointLine[];
  clipping_planes?: ViewpointClippingPlane[];
  bitmaps?: ViewpointBitmap[];
}
```

### Camera Types

```typescript
interface ViewpointCamera {
  camera_view_point: ViewpointVector;
  camera_direction: ViewpointVector;
  camera_up_vector: ViewpointVector;
  aspect_ratio?: number;
}

interface ViewpointPerspectiveCamera extends ViewpointCamera {
  field_of_view: number;
}

interface ViewpointOrthogonalCamera extends ViewpointCamera {
  view_to_world_scale: number;
}

interface ViewpointVector {
  x: number;
  y: number;
  z: number;
}
```

### Component Types

```typescript
interface ViewpointComponents {
  selection?: ViewpointComponent[];
  coloring?: ViewpointColoring[];
  visibility?: ViewpointVisibility;
}

interface ViewpointComponent {
  ifc_guid: string;
  authoring_tool_id?: string;
  originating_system?: string;
}

interface ViewpointVisibility {
  default_visibility: boolean;
  exceptions?: ViewpointComponent[];
  view_setup_hints?: {
    spaces_visible?: boolean;
    space_boundaries_visible?: boolean;
    openings_visible?: boolean;
  };
}

interface ViewpointColoring {
  color: string; // hex
  components: ViewpointComponent[];
}
```

### Supporting Types

```typescript
interface ViewpointClippingPlane {
  location: ViewpointVector;
  direction: ViewpointVector;
}

interface ViewpointLine {
  start_point: ViewpointVector;
  end_point: ViewpointVector;
}

interface ViewpointSnapshot {
  snapshot_type: "png" | "jpg";
  snapshot_data: string; // base64
}

interface ViewpointBitmap {
  bitmap_type: string;
  bitmap_data: string; // base64
  location: ViewpointVector;
  normal: ViewpointVector;
  up: ViewpointVector;
  height: number;
}
```
