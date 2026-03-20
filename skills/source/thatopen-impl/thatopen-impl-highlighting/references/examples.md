# Highlighting Examples

## Basic Click-to-Select

Minimal setup for click selection with default yellow highlight:

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

// Assumes world is already set up (see thatopen-impl-viewer)
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// That's it — clicking elements now highlights them in yellow (#BCF124)
```

## Custom Selection Color

```typescript
import * as THREE from "three";

const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({
  world,
  selectionColor: new THREE.Color("#3498db"), // Blue selection
});
```

## Multi-Select with Shift Key

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Hold Shift to add to selection
highlighter.multiple = "shiftKey";

// Or use Ctrl
// highlighter.multiple = "ctrlKey";
```

## Toggle Selection (Click to Select/Deselect)

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Clicking the same element again deselects it
highlighter.autoToggle.add("select");
```

## Multiple Highlight Styles

```typescript
import * as THREE from "three";

const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Warning style — semi-transparent red
highlighter.styles.set("warning", {
  color: new THREE.Color("#FF0000"),
  opacity: 0.5,
});

// Info style — semi-transparent blue
highlighter.styles.set("info", {
  color: new THREE.Color("#2196F3"),
  opacity: 0.3,
});

// Invisible tracking style — no visual change
highlighter.styles.set("tracked", null);

// Apply styles programmatically
const warningItems: ModelIdMap = new Map();
warningItems.set(modelId, new Set([101, 102]));
await highlighter.highlightByID("warning", warningItems);

const infoItems: ModelIdMap = new Map();
infoItems.set(modelId, new Set([201, 202, 203]));
await highlighter.highlightByID("info", infoItems);
```

## Zoom to Selection

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Global: every highlight zooms the camera
highlighter.zoomToSelection = true;

// Or per-call:
await highlighter.highlight("select", true, true);
```

## Programmatic Selection by Element IDs

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Build a ModelIdMap
const items: ModelIdMap = new Map();
items.set("model-uuid-abc123", new Set([5001, 5002, 5003]));

// Highlight without removing previous selection
await highlighter.highlightByID("select", items, false);

// Read the current selection
const selected = highlighter.selection["select"];
for (const [modelId, elementIds] of selected) {
  console.log(`Model ${modelId}: elements`, [...elementIds]);
}
```

## Restrict Selectable Elements

Only allow specific elements to be selected:

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Only walls and slabs can be selected
const selectableItems: ModelIdMap = new Map();
selectableItems.set(modelId, new Set([wallId1, wallId2, slabId1]));
highlighter.selectable["select"] = selectableItems;

// Clicking on doors, windows, etc. will have no effect
```

## Hover Effect with Hoverer

```typescript
const hoverer = components.get(OBCF.Hoverer);

hoverer.onHoverStarted.add((data) => {
  // Update UI with hovered element info
  statusBar.textContent = `Hovering: ${data}`;
});

hoverer.onHoverEnded.add(() => {
  statusBar.textContent = "";
});
```

## Post-Processing Outlines with Outliner

Requires PostproductionRenderer:

```typescript
import * as THREE from "three";
import * as OBCF from "@thatopen/components-front";

// World MUST use PostproductionRenderer
world.renderer.postproduction.enabled = true;

// Setup Highlighter first
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// Setup Outliner to track the "select" style
const outliner = components.get(OBCF.Outliner);
outliner.styles.add("select");

// Configure outline appearance
outliner.color = new THREE.Color("#FF6600");
outliner.thickness = 3;
outliner.fillColor = new THREE.Color("#FF6600");
outliner.fillOpacity = 0.05;

// Now when elements are highlighted with "select" style,
// they also get a post-processing outline
```

## Manual Outline Management

```typescript
const outliner = components.get(OBCF.Outliner);

// Add outlines to specific elements
const items: ModelIdMap = new Map();
items.set(modelId, new Set([elementId]));
outliner.addItems(items);

// Remove outlines from specific elements
outliner.removeItems(items);

// Remove all outlines
outliner.clean();
```

## Convert Selection to THREE.Mesh (Mesher)

```typescript
const mesher = components.get(OBCF.Mesher);

// Get selected elements as standard meshes
const items: ModelIdMap = new Map();
items.set(modelId, new Set([elementId1, elementId2]));

const meshes = mesher.get(items);

// Add to scene for custom rendering
for (const mesh of meshes) {
  world.scene.three.add(mesh);
}

// Clean up when done — YOU must dispose manually
for (const mesh of meshes) {
  mesh.geometry.dispose();
  if (Array.isArray(mesh.material)) {
    mesh.material.forEach((m) => m.dispose());
  } else {
    mesh.material.dispose();
  }
  world.scene.three.remove(mesh);
}
```

## Full Workflow: Highlight + Outline + Hover

```typescript
import * as THREE from "three";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

// Assumes PostproductionRenderer world is set up
world.renderer.postproduction.enabled = true;

// 1. Highlighter with multi-select
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });
highlighter.multiple = "shiftKey";
highlighter.zoomToSelection = true;
highlighter.autoToggle.add("select");

// 2. Custom warning style
highlighter.styles.set("clash", {
  color: new THREE.Color("#FF0000"),
  opacity: 0.7,
});

// 3. Outliner for both styles
const outliner = components.get(OBCF.Outliner);
outliner.styles.add("select");
outliner.styles.add("clash");
outliner.color = new THREE.Color("#FFFFFF");
outliner.thickness = 2;

// 4. Hoverer for feedback
const hoverer = components.get(OBCF.Hoverer);
hoverer.onHoverStarted.add(() => {
  container.style.cursor = "pointer";
});
hoverer.onHoverEnded.add(() => {
  container.style.cursor = "default";
});

// Now: Shift+click to multi-select with outlines,
// hover shows cursor change, clash style available for
// programmatic highlighting of clash results
```
