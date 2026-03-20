# Anti-Patterns — Clipping Planes & Floor Plans

## 1. Using v2 Plans Component

**WRONG** — `Plans` does not exist in v3:
```typescript
// BROKEN — Plans is removed in v3
import { Plans } from "@thatopen/components";
const plans = components.get(Plans);
plans.create({ name: "Floor 1", ... });
plans.goTo("Floor 1");
```

**RIGHT** — Use Views + ClipStyler:
```typescript
import * as OBC from "@thatopen/components";
import * as OBF from "@thatopen/components-front";

const views = components.get(OBC.Views);
views.world = world;
const view = views.create(normal, point, { id: "floor-1" });

const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;
clipStyler.createFromView(view, { items: { All: { style: "Default" } } });

views.open("floor-1");
```

---

## 2. Using Standalone ClipEdges from v2

**WRONG** — Standalone ClipEdges is removed:
```typescript
// BROKEN — ClipEdges is not a standalone component in v3
const clipEdges = components.get(ClipEdges);
clipEdges.styles.create("Default", [], material);
```

**RIGHT** — ClipEdges are managed through ClipStyler:
```typescript
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;
clipStyler.styles.set("Default", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
});
clipStyler.createFromClipping(planeId, {
  items: { All: { style: "Default" } },
});
```

---

## 3. Forgetting to Set ClipStyler World

**WRONG** — Missing world assignment:
```typescript
const clipStyler = components.get(OBF.ClipStyler);
// clipStyler.world = world;  // MISSING!
clipStyler.createFromClipping(planeId, { ... });
// Edges created but not added to any scene
```

**RIGHT** — ALWAYS set world first:
```typescript
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;  // REQUIRED
clipStyler.createFromClipping(planeId, { ... });
```

---

## 4. Creating Styled Edges Before Defining Styles

**WRONG** — Style referenced but not defined:
```typescript
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;

// Style "Section" not yet defined!
clipStyler.createFromClipping(planeId, {
  items: { All: { style: "Section" } },
});

// Too late
clipStyler.styles.set("Section", { ... });
```

**RIGHT** — Define styles first:
```typescript
const clipStyler = components.get(OBF.ClipStyler);
clipStyler.world = world;

clipStyler.styles.set("Section", {
  linesMaterial: new LineMaterial({ color: 0x000000, linewidth: 2 }),
});

clipStyler.createFromClipping(planeId, {
  items: { All: { style: "Section" } },
});
```

---

## 5. Forgetting to Enable Clipper

**WRONG** — Clipper disabled by default:
```typescript
const clipper = components.get(OBC.Clipper);
// clipper.enabled is false by default!
clipper.create(world);  // Does nothing
```

**RIGHT** — Enable before use:
```typescript
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;
clipper.create(world);
```

---

## 6. Using LineBasicMaterial for Edge Lines

**WRONG** — `linewidth` has no effect with `LineBasicMaterial`:
```typescript
clipStyler.styles.set("Default", {
  linesMaterial: new THREE.LineBasicMaterial({
    color: 0x000000,
    linewidth: 3,  // IGNORED by WebGL
  }),
});
```

**RIGHT** — Use `LineMaterial` from Three.js examples:
```typescript
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial.js";

clipStyler.styles.set("Default", {
  linesMaterial: new LineMaterial({
    color: 0x000000,
    linewidth: 3,  // Works correctly
  }),
});
```

---

## 7. Manually Positioning Camera for Floor Plans

**WRONG** — Manual camera setup when using Views:
```typescript
views.open("floor-1");
// Redundant — views.open() already handles camera
world.camera.set("Plan");
await world.camera.controls.setLookAt(x, y, z, tx, ty, tz);
```

**RIGHT** — Let `views.open()` handle camera positioning:
```typescript
views.open("floor-1");
// Only set projection if needed
await world.camera.projection.set("Orthographic");
```

`views.open()` automatically switches to Plan mode and positions the camera
relative to the view plane. NEVER manually reposition the camera after opening
a view.

---

## 8. Not Disposing Clipping Resources

**WRONG** — Memory leak:
```typescript
// Creating planes without cleanup
for (const floor of floors) {
  clipper.createFromNormalAndCoplanarPoint(world, normal, point);
}
// No disposal — planes and edge geometry accumulate
```

**RIGHT** — Clean up when done:
```typescript
// Remove all clipping planes
clipper.deleteAll();

// Or dispose the entire ClipStyler (clears styles and edges)
clipStyler.dispose();
```

---

## 9. Forgetting to Close Views Before Switching

**WRONG** — Opening another view without closing:
```typescript
views.open("floor-1");
// ... user clicks another floor
views.open("floor-2");  // Previous view may not be properly deactivated
```

**RIGHT** — Close first, or rely on Views auto-management:
```typescript
views.close();           // Close current view
views.open("floor-2");   // Open new one
```

Note: `views.open()` ensures only one view per world is active, but
explicitly closing first is clearer and avoids edge cases.

---

## 10. Creating Planes Without a Loaded Model

**WRONG** — Interactive plane creation on empty scene:
```typescript
const clipper = components.get(OBC.Clipper);
clipper.enabled = true;
clipper.create(world);  // Returns null — no geometry to raycast against
```

**RIGHT** — Load geometry first, or use programmatic creation:
```typescript
// Option A: Load model first
await fragments.load(data);
clipper.create(world);  // Now raycasts hit geometry

// Option B: Use programmatic creation (no raycast needed)
clipper.createFromNormalAndCoplanarPoint(world, normal, point);
```
