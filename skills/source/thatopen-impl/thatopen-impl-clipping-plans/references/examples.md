# Examples — Clipping Planes & Floor Plans

## 1. Basic Interactive Clipping

Minimal setup to let users create clipping planes by double-clicking on
geometry.

```typescript
import * as OBC from "@thatopen/components";

// Assume components and world are set up (see thatopen-impl-viewer)
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;

// Double-click to create a plane at intersection point
container.ondblclick = () => {
  if (clipper.enabled) {
    clipper.create(world);
  }
};

// Delete key removes the plane under cursor
window.onkeydown = (event) => {
  if (event.code === "Delete" || event.code === "Backspace") {
    clipper.delete(world);
  }
};
```

## 2. Programmatic Section Cut

Create a horizontal section at a specific elevation without user interaction.

```typescript
import * as OBC from "@thatopen/components";
import * as THREE from "three";

const clipper = components.get(OBC.Clipper);
clipper.enabled = true;

// Horizontal cut at y=3.0m, normal pointing down
const planeId = clipper.createFromNormalAndCoplanarPoint(
  world,
  new THREE.Vector3(0, -1, 0),
  new THREE.Vector3(0, 3.0, 0),
);

// Hide the plane helper visual
clipper.visible = false;

// Later: remove the plane
await clipper.delete(world, planeId);
```

## 3. Styled Section with ClipStyler

Add edge outlines to a programmatic clipping plane.

```typescript
import * as OBC from "@thatopen/components";
import * as OBF from "@thatopen/components-front";
import * as THREE from "three";
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial.js";

// Create clipper plane
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;

const planeId = clipper.createFromNormalAndCoplanarPoint(
  world,
  new THREE.Vector3(0, -1, 0),
  new THREE.Vector3(0, 3.0, 0),
);

// Set up ClipStyler
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;

// Define a style
clipStyler.styles.set("Section", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
  fillsMaterial: new THREE.MeshBasicMaterial({
    color: 0xe0e0e0,
    side: THREE.DoubleSide,
  }),
});

// Link styled edges to the clipping plane
clipStyler.createFromClipping(planeId, {
  items: { All: { style: "Section" } },
});
```

## 4. Auto-Style New Planes

Automatically add styled edges whenever a new clipping plane is created.

```typescript
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;

const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;

clipStyler.styles.set("Default", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
});

// React to new planes
clipper.list.onItemSet.add(({ key }) => {
  clipStyler.createFromClipping(key, {
    items: { All: { style: "Default" } },
  });
});

// Now any user-created or programmatic plane gets edges
container.ondblclick = () => clipper.create(world);
```

## 5. Floor Plan from IFC Storeys

Generate floor plan views automatically from the IFC spatial structure.

```typescript
import * as OBC from "@thatopen/components";
import * as OBF from "@thatopen/components-front";
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial.js";
import * as THREE from "three";

// Set up Views
const views = components.get(OBC.Views);
views.world = world;

// Create views from IFC storeys (one per floor)
const storeyViews = await views.createFromIfcStoreys({
  offset: 0.25,  // 25cm above slab level
});

// Set up ClipStyler
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;

clipStyler.styles.set("Plan", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
  fillsMaterial: new THREE.MeshBasicMaterial({
    color: 0xf5f5f5,
    side: THREE.DoubleSide,
  }),
});

// Add styled edges to each storey view
for (const [id, view] of views.list) {
  clipStyler.createFromView(view, {
    items: { All: { style: "Plan" } },
  });
}

// Open the ground floor plan
views.open("ground-floor");
await world.camera.projection.set("Orthographic");
```

## 6. Per-Category Floor Plan Styling

Different styles for walls, windows, and structural elements.

```typescript
// Define multiple styles
clipStyler.styles.set("Walls", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 3 }),
  fillsMaterial: new THREE.MeshBasicMaterial({ color: 0xcccccc, side: 2 }),
});

clipStyler.styles.set("Windows", {
  linesMaterial: new LineMaterial({ color: 0x0066ff, linewidth: 1 }),
  fillsMaterial: new THREE.MeshBasicMaterial({ color: 0xccddff, side: 2 }),
});

clipStyler.styles.set("Structure", {
  linesMaterial: new LineMaterial({ color: 0x333333, linewidth: 2 }),
  fillsMaterial: new THREE.MeshBasicMaterial({ color: 0xaaaaaa, side: 2 }),
});

// Apply per-category
clipStyler.createFromView(view, {
  items: {
    WallGroup: {
      style: "Walls",
      data: { "BuildingElement.Class": ["IFCWALL", "IFCWALLSTANDARDCASE"] },
    },
    WindowGroup: {
      style: "Windows",
      data: { "BuildingElement.Class": ["IFCWINDOW"] },
    },
    StructureGroup: {
      style: "Structure",
      data: { "BuildingElement.Class": ["IFCSLAB", "IFCCOLUMN", "IFCBEAM"] },
    },
  },
});
```

## 7. Elevation Views

Create front/back/left/right elevation views from the model bounding box.

```typescript
const views = components.get(OBC.Views);
views.world = world;

const elevations = await views.createElevations({
  combine: true,  // single bounding box for all models
});

// Open the front elevation
views.open("front");
await world.camera.projection.set("Orthographic");
```

## 8. Floor Plan Toggle UI

Switch between 3D view and floor plans.

```typescript
function openFloorPlan(viewId: string) {
  views.open(viewId);
  world.camera.projection.set("Orthographic");
  clipStyler.visible = true;
}

function returnTo3D() {
  views.close();
  world.camera.projection.set("Perspective");
  world.camera.set("Orbit");
  clipStyler.visible = false;
}

// Wire to UI buttons
floorPlanButton.onclick = () => openFloorPlan("ground-floor");
threeDButton.onclick = () => returnTo3D();
```

## 9. Vertical Section Cut

Create a vertical section through a building.

```typescript
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;

// Vertical plane cutting along X axis
const planeId = clipper.createFromNormalAndCoplanarPoint(
  world,
  new THREE.Vector3(1, 0, 0),   // normal pointing in +X
  new THREE.Vector3(5.0, 0, 0), // at x=5.0m
);

// Add styled edges
clipStyler.createFromClipping(planeId, {
  items: { All: { style: "Section" } },
});
```

## 10. Cleanup Pattern

Proper disposal of all clipping resources.

```typescript
// Remove all clipping planes
clipper.deleteAll();

// Dispose ClipStyler (clears all styles and edges)
clipStyler.dispose();

// Or dispose everything at once via components
components.dispose();
```
