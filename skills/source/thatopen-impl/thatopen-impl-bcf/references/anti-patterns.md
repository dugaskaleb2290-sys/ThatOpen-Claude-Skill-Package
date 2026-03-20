# BCF & Viewpoints — Anti-Patterns

## AP-1: Missing setup() Before Operations

**Wrong:**
```typescript
const bcfTopics = components.get(OBC.BCFTopics);
// No setup() call
const topic = bcfTopics.create({ title: "Issue" });
// author is empty string, no extensions configured
const blob = await bcfTopics.export();
// version is empty string — produces invalid BCF
```

**Correct:**
```typescript
const bcfTopics = components.get(OBC.BCFTopics);
bcfTopics.setup({
  version: "3",
  author: "user@example.com",
  types: new Set(["Issue"]),
  statuses: new Set(["Active"]),
});
const topic = bcfTopics.create({ title: "Issue" });
const blob = await bcfTopics.export();
```

ALWAYS call `setup()` with at least `version` and `author` before any
BCF operations. Without it, exported BCF files lack version info, author
attribution, and extension definitions.

---

## AP-2: Wrong BCF Version String

**Wrong:**
```typescript
bcfTopics.setup({ version: "3.0" });  // "3.0" is not a valid BCFVersion
bcfTopics.setup({ version: "v2.1" }); // "v2.1" is not valid
bcfTopics.setup({ version: 3 });      // number, not string
```

**Correct:**
```typescript
bcfTopics.setup({ version: "3" });    // BCF 3.0
bcfTopics.setup({ version: "2.1" });  // BCF 2.1
```

The `BCFVersion` type is `"2.1" | "3"`. NEVER use `"3.0"`, `"v2.1"`,
or numeric values.

---

## AP-3: Missing Viewpoints World

**Wrong:**
```typescript
const viewpoints = components.get(OBC.Viewpoints);
// No world set
const vp = viewpoints.create();
await vp.updateCamera(); // Fails — no world to read camera from
vp.takeSnapshot();       // Fails — no world renderer to capture
```

**Correct:**
```typescript
const viewpoints = components.get(OBC.Viewpoints);
viewpoints.world = world; // Set before creating viewpoints
const vp = viewpoints.create();
await vp.updateCamera();
vp.takeSnapshot();
```

ALWAYS set `viewpoints.world` before creating viewpoints or calling
`updateCamera()` / `takeSnapshot()`.

---

## AP-4: Storing Object References Instead of GUIDs

**Wrong:**
```typescript
const topic = bcfTopics.create({ title: "Issue" });
const viewpoint = viewpoints.create();

// Storing object reference — memory leak risk
myApp.savedViewpoint = viewpoint;
myApp.savedTopic = topic;

// Object reference on topic — breaks serialization
topic.viewpoints.add(viewpoint); // Wrong type — expects string GUID
```

**Correct:**
```typescript
const topic = bcfTopics.create({ title: "Issue" });
const viewpoint = viewpoints.create();

// Store GUIDs
myApp.savedViewpointGuid = viewpoint.guid;
myApp.savedTopicGuid = topic.guid;

// Link via GUID
topic.viewpoints.add(viewpoint.guid);

// Look up later
const vp = viewpoints.list.get(myApp.savedViewpointGuid);
const tp = bcfTopics.list.get(myApp.savedTopicGuid);
```

ALWAYS use GUIDs for cross-references. The DataSet/DataMap collections
manage lifecycle; external object references prevent garbage collection.

---

## AP-5: Forgetting updateCamera() on New Viewpoints

**Wrong:**
```typescript
const vp = viewpoints.create();
vp.title = "Important view";
topic.viewpoints.add(vp.guid);
// Viewpoint has no camera data — go() will not position camera
```

**Correct:**
```typescript
const vp = viewpoints.create();
vp.title = "Important view";
await vp.updateCamera(true); // Capture camera + snapshot
topic.viewpoints.add(vp.guid);
```

New viewpoints have no camera data by default. ALWAYS call
`updateCamera()` after creation to capture the current camera state.

---

## AP-6: Strict Mode with Empty Extensions

**Wrong:**
```typescript
bcfTopics.setup({
  version: "3",
  author: "user@example.com",
  strict: true,
  // types, statuses, priorities all empty (default)
});

// Throws — "Issue" is not in empty types Set
const topic = bcfTopics.create({ type: "Issue" });
```

**Correct:**
```typescript
bcfTopics.setup({
  version: "3",
  author: "user@example.com",
  strict: true,
  types: new Set(["Issue", "Request"]),
  statuses: new Set(["Active", "Closed"]),
  priorities: new Set(["Critical", "Normal"]),
});

const topic = bcfTopics.create({ type: "Issue" }); // Works
```

When `strict: true`, ALWAYS populate the extension Sets with all
allowed values. Empty Sets mean nothing is allowed.

---

## AP-7: Importing Without Fallback Version

**Wrong:**
```typescript
bcfTopics.setup({ version: "3", author: "user@example.com" });
// fallbackVersionOnImport defaults to null
const data = new Uint8Array(await file.arrayBuffer());
await bcfTopics.load(data);
// If the BCF file has no bcf.version XML, parsing may fail or produce
// unexpected results
```

**Correct:**
```typescript
bcfTopics.setup({
  version: "3",
  author: "user@example.com",
  fallbackVersionOnImport: "2.1", // Handle version-less files
  ignoreIncompleteTopicsOnImport: true, // Skip malformed topics
});
const data = new Uint8Array(await file.arrayBuffer());
const { topics } = await bcfTopics.load(data);
```

ALWAYS set `fallbackVersionOnImport` when importing BCF files from
external tools. Not all BCF exporters include version metadata.

---

## AP-8: Not Disposing BCF Components

**Wrong:**
```typescript
// App shutdown — BCFTopics and Viewpoints not disposed
// DataMaps with topics, viewpoints, snapshots remain in memory
```

**Correct:**
```typescript
// Dispose explicitly
bcfTopics.dispose();
viewpoints.dispose();

// Or via components (preferred)
components.dispose();
```

ALWAYS dispose BCF components on cleanup. The `snapshots` DataMap in
Viewpoints holds binary image data that can consume significant memory.

---

## AP-9: Direct Assignment for Reactive UI

**Wrong:**
```typescript
// UI table bound to topic events
topic.title = "Updated title";  // No event fired — UI does not update
topic.status = "Resolved";      // No event fired — UI does not update
```

**Correct:**
```typescript
// Use set() for reactive updates
topic.set({ title: "Updated title", status: "Resolved" });
// Events fire — UI updates
```

Use `set()` when UI components listen for topic changes. Direct property
assignment is silent. This matters for reactive frameworks and the
`@thatopen/ui-obc` table components.

---

## AP-10: Mixing BCF Versions in One Session

**Wrong:**
```typescript
// Import BCF 2.1 file
bcfTopics.setup({ version: "2.1", author: "user@example.com" });
await bcfTopics.load(bcf21Data);

// Change version mid-session and export
bcfTopics.setup({ version: "3", author: "user@example.com" });
await bcfTopics.export();
// Topics originally from 2.1 are now exported as 3.0
// Viewpoint XML serialization format changes
```

**Correct:**
```typescript
// Keep consistent version throughout a workflow
bcfTopics.setup({ version: "3", author: "user@example.com" });

// Import handles version detection automatically
await bcfTopics.load(bcf21Data); // Reads version from bcf.version XML

// Export uses the configured version
await bcfTopics.export(); // Exports as v3.0
```

Be intentional about version changes. The `load()` method reads the
BCF file's own version. The `export()` method uses the configured
`config.version`. Changing version between import and export converts
the data, which may lose version-specific features.

---

## AP-11: Creating Topics Without Required Fields

**Wrong:**
```typescript
const topic = bcfTopics.create();
// title = "BCF Topic" (generic default)
// type = "Issue" (may not be in config types)
// No description, no assignee
```

**Correct:**
```typescript
const topic = bcfTopics.create({
  title: "Specific, descriptive issue title",
  type: "Issue",
  priority: "Major",
  assignedTo: "responsible@company.com",
  description: "Clear description of what needs to be addressed",
});
```

ALWAYS provide meaningful `title`, `type`, and `description` when
creating topics. Default values produce generic, unhelpful BCF entries.
