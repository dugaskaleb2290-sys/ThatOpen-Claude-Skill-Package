# BCF & Viewpoints — Examples

## 1. Basic Setup and Topic Creation

```typescript
import * as OBC from "@thatopen/components";

// Get components
const bcfTopics = components.get(OBC.BCFTopics);
const viewpoints = components.get(OBC.Viewpoints);

// Configure BCFTopics — ALWAYS do this first
bcfTopics.setup({
  version: "3",
  author: "architect@company.com",
  types: new Set(["Issue", "Request", "Comment", "Fault"]),
  statuses: new Set(["Active", "InProgress", "Resolved", "Closed"]),
  priorities: new Set(["Critical", "Major", "Normal", "Minor"]),
  labels: new Set(["Architecture", "Structure", "MEP", "Safety"]),
  stages: new Set(["Schematic Design", "Design Development", "Construction"]),
  users: new Set(["architect@company.com", "engineer@company.com"]),
});

// Set viewpoints world
viewpoints.world = world;

// Create a topic
const topic = bcfTopics.create({
  title: "Missing fire rating on Level 3 wall",
  type: "Issue",
  priority: "Critical",
  status: "Active",
  assignedTo: "engineer@company.com",
  description: "Wall W-301 between grid B-C lacks required 2-hour fire rating",
  stage: "Design Development",
});

// Add labels
topic.labels.add("Architecture");
topic.labels.add("Safety");

// Add a comment
topic.createComment("Detected during model review session 2024-03-15");
```

## 2. Creating a Viewpoint with Snapshot

```typescript
// Create viewpoint from current camera state
const viewpoint = viewpoints.create();
viewpoint.title = "View of wall W-301";

// Capture camera position and take snapshot
await viewpoint.updateCamera(true); // true = also take snapshot

// Link viewpoint to topic
topic.viewpoints.add(viewpoint.guid);

// Add selected elements to viewpoint
viewpoint.selectionComponents.add(
  "2O2Fr$t4X7Zf8NOew3FLOH", // Wall W-301 IFC GUID
);
```

## 3. Creating a Viewpoint with Element Selection

```typescript
const viewpoint = viewpoints.create();
viewpoint.title = "Highlighted clash area";

// Capture camera
await viewpoint.updateCamera();

// Add specific elements by IFC GUID
viewpoint.selectionComponents.add(
  "3V$FMCDUfCoPwUaHMPfteW",
  "1fIVuvFffDJRV_SJESOtCZ",
);

// Add elements by category using ItemsFinder
const finder = components.get(OBC.ItemsFinder);
const doors = await finder.getItems([{ categories: [/DOOR/] }]);
const fragments = components.get(OBC.FragmentsManager);
const guids = await fragments.modelIdMapToGuids(doors);
viewpoint.selectionComponents.add(...guids);

// Set visibility: hide everything except selected
viewpoint.defaultVisibility = false;
viewpoint.exceptionComponents.add(...guids);
```

## 4. Applying a Viewpoint

```typescript
// Navigate camera to viewpoint
const vp = viewpoints.list.get(viewpointGuid);
if (vp) {
  await vp.go(); // Sets camera, visibility, colorization, clipping
}
```

## 5. Export BCF

```typescript
// Export all topics
const blob = await bcfTopics.export();
downloadBlob(blob, "project-issues.bcf");

// Export only active topics
const activeTopics = [...bcfTopics.list.values()]
  .filter(t => t.status === "Active");
const activeBlob = await bcfTopics.export(activeTopics);
downloadBlob(activeBlob, "active-issues.bcf");

// Helper function
function downloadBlob(blob: Blob, filename: string) {
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.href = url;
  link.download = filename;
  link.click();
  URL.revokeObjectURL(url);
}
```

## 6. Import BCF

```typescript
// From file input
async function importBCF(file: File) {
  const data = new Uint8Array(await file.arrayBuffer());
  const { topics, viewpoints: importedVPs } = await bcfTopics.load(data);
  console.log(`Imported ${topics.length} topics, ${importedVPs.length} viewpoints`);
  return { topics, viewpoints: importedVPs };
}

// With file picker
const input = document.createElement("input");
input.type = "file";
input.accept = ".bcf,.bcfzip";
input.addEventListener("change", async () => {
  const file = input.files?.[0];
  if (file) await importBCF(file);
});
input.click();
```

## 7. Listen for BCF Import Events

```typescript
bcfTopics.onBCFImported.add((importedTopics) => {
  console.log(`${importedTopics.length} topics imported`);
  for (const topic of importedTopics) {
    console.log(`- [${topic.type}] ${topic.title} (${topic.status})`);
  }
});
```

## 8. Auto-link Viewpoints to Topics

```typescript
// Automatically create a topic for each new viewpoint
viewpoints.list.onItemSet.add(({ value: vp }) => {
  const topic = bcfTopics.create({
    title: vp.title || "New viewpoint",
    type: "Comment",
  });
  topic.viewpoints.add(vp.guid);
});
```

## 9. Updating Topic Properties Reactively

```typescript
// Use set() for reactive updates (fires events for UI listeners)
topic.set({
  status: "Resolved",
  priority: "Minor",
});

// Add a resolution comment
topic.createComment("Fire rating specification added to wall assembly");
```

## 10. BCF v2.1 Export

```typescript
// Configure for BCF 2.1 compatibility
bcfTopics.setup({
  version: "2.1",
  author: "user@example.com",
  types: new Set(["Error", "Warning", "Info"]),
  statuses: new Set(["Open", "Closed"]),
  priorities: new Set(["High", "Normal", "Low"]),
});

const blob = await bcfTopics.export();
```

## 11. Strict Mode Validation

```typescript
bcfTopics.setup({
  version: "3",
  author: "user@example.com",
  types: new Set(["Issue", "Request"]),
  statuses: new Set(["Active", "Closed"]),
  strict: true, // Enforce validation
});

// This works
const topic = bcfTopics.create({ type: "Issue", status: "Active" });

// This would throw — "Fault" is not in the configured types
// const bad = bcfTopics.create({ type: "Fault" });
```

## 12. Document References

```typescript
// Add external document reference
const docGuid = crypto.randomUUID();
bcfTopics.documents.set(docGuid, {
  type: "external",
  url: "https://docs.example.com/specs/fire-rating.pdf",
  description: "Fire rating specification document",
});
topic.documentReferences.add(docGuid);

// Add internal document (embedded in BCF export)
const internalGuid = crypto.randomUUID();
bcfTopics.documents.set(internalGuid, {
  type: "internal",
  fileName: "screenshot.png",
  data: screenshotData, // Uint8Array
  description: "Screenshot of the issue area",
});
topic.documentReferences.add(internalGuid);
```

## 13. Viewpoint with Component Colors

```typescript
const viewpoint = viewpoints.create();
await viewpoint.updateCamera();

// Color specific components
viewpoint.componentColors.set("#ff0000", [
  "2O2Fr$t4X7Zf8NOew3FLOH", // Red for problematic elements
]);
viewpoint.componentColors.set("#00ff00", [
  "3V$FMCDUfCoPwUaHMPfteW", // Green for compliant elements
]);

// Apply colorization
viewpoint.setColorizationState(true);
```

## 14. Viewpoint with Clipping Planes

```typescript
const viewpoint = viewpoints.create();
await viewpoint.updateCamera();

// Capture current clipping planes
viewpoint.updateClippingPlanes();

// Apply viewpoint (restores clipping state)
await viewpoint.go();
```

## 15. Complete Workflow: Issue Tracking

```typescript
import * as OBC from "@thatopen/components";

// Setup
const bcfTopics = components.get(OBC.BCFTopics);
const viewpoints = components.get(OBC.Viewpoints);

bcfTopics.setup({
  version: "3",
  author: "coordinator@company.com",
  types: new Set(["Clash", "Issue", "Request"]),
  statuses: new Set(["Open", "InProgress", "Resolved", "Closed"]),
  priorities: new Set(["Critical", "Major", "Normal", "Minor"]),
  users: new Set(["coordinator@company.com", "architect@company.com"]),
});
viewpoints.world = world;

// 1. Detect issue, create viewpoint
const vp = viewpoints.create();
vp.title = "Clash at grid intersection B-3";
await vp.updateCamera(true);

// 2. Create topic
const topic = bcfTopics.create({
  title: "Steel beam clashes with HVAC duct at B-3",
  type: "Clash",
  priority: "Critical",
  assignedTo: "architect@company.com",
  description: "Beam HEB300 at +8.400m intersects with DN400 supply duct",
});
topic.viewpoints.add(vp.guid);
topic.createComment("Detected during coordination check. Duct must be rerouted.");

// 3. Export for sharing
const blob = await bcfTopics.export([topic]);
downloadBlob(blob, "clash-B3.bcf");

// 4. Later: resolve
topic.set({ status: "Resolved" });
topic.createComment("Duct rerouted via alternative path above beam");

// 5. Update extensions to reflect current state
bcfTopics.updateExtensions();
```

## 16. Cleanup

```typescript
// Dispose individual components
bcfTopics.dispose();
viewpoints.dispose();

// Or let components handle everything
components.dispose();
```
