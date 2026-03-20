# Navigation Anti-Patterns

## AP-1: FirstPerson with Orthographic Projection

**WRONG** — FirstPerson mode silently falls back to Orbit when orthographic
is active:

```typescript
// BAD: No validation before switching
await world.camera.projection.set("Orthographic");
world.camera.set("FirstPerson"); // Silently reverts to Orbit!
```

**CORRECT** — ALWAYS validate projection compatibility:

```typescript
// GOOD: Validate before switching
if (world.camera.projection.current !== "Orthographic") {
  world.camera.set("FirstPerson");
} else {
  console.warn("Switch to Perspective first.");
}
```

---

## AP-2: Setting Camera Position Directly on Three.js Object

**WRONG** — camera-controls overwrites direct position changes on the next
frame:

```typescript
// BAD: Position is overwritten by camera-controls
world.camera.three.position.set(10, 20, 30);
world.camera.three.lookAt(0, 0, 0);
```

**CORRECT** — ALWAYS use camera-controls methods:

```typescript
// GOOD: camera-controls manages position internally
await world.camera.controls.setLookAt(10, 20, 30, 0, 0, 0);
```

---

## AP-3: Forgetting to Await Projection Changes

**WRONG** — projection switching returns a Promise; not awaiting it causes
race conditions:

```typescript
// BAD: Projection may not have finished switching
world.camera.projection.set("Orthographic");
world.camera.set("Plan"); // May run before projection is ready
```

**CORRECT** — ALWAYS await projection changes:

```typescript
// GOOD: Wait for projection to complete
await world.camera.projection.set("Orthographic");
world.camera.set("Plan");
```

---

## AP-4: Not Disabling User Input During Animations

**WRONG** — User can interfere with programmatic camera movement:

```typescript
// BAD: User input can conflict with animation
await world.camera.controls.setLookAt(x, y, z, tx, ty, tz, true);
```

**CORRECT** — Lock controls during animation:

```typescript
// GOOD: Prevent user interference
world.camera.setUserInput(false);
await world.camera.controls.setLookAt(x, y, z, tx, ty, tz, true);
world.camera.setUserInput(true);
```

---

## AP-5: Not Listening to Projection Changes

**WRONG** — Components that depend on projection type become out of sync:

```typescript
// BAD: Grid fade never updates when projection changes
grid.fade = true; // Only set once at init
```

**CORRECT** — ALWAYS listen for projection changes:

```typescript
// GOOD: React to projection changes
world.camera.projection.onChanged.add(() => {
  grid.fade = world.camera.projection.current === "Perspective";
});
```

---

## AP-6: Using Wrong Mode for Floor Plans

**WRONG** — Using Orbit mode for floor plans allows unwanted rotation:

```typescript
// BAD: User can rotate out of the top-down view
world.camera.set("Orbit");
await world.camera.projection.set("Orthographic");
await world.camera.controls.setLookAt(0, 100, 0, 0, 0, 0);
// User rotates and loses the floor plan view
```

**CORRECT** — ALWAYS use Plan mode for floor plans:

```typescript
// GOOD: Plan mode locks rotation
world.camera.set("Plan");
await world.camera.projection.set("Orthographic");
await world.camera.controls.setLookAt(0, 100, 0, 0, 0, 0);
// Rotation is disabled — view stays top-down
```

---

## AP-7: Not Updating Fragments on Camera Change

**WRONG** — Fragment LOD and culling become stale:

```typescript
// BAD: No camera update listener
// Fragments render with wrong LOD as camera moves
```

**CORRECT** — ALWAYS connect camera updates to fragment system:

```typescript
// GOOD: Update fragments when camera moves
world.camera.controls.addEventListener("update", () => {
  fragments.core.update();
});

world.onCameraChanged.add((camera) => {
  for (const [, model] of fragments.list) {
    model.useCamera(camera.three);
  }
  fragments.core.update(true);
});
```

---

## AP-8: Modifying camera-controls Settings Without Understanding Mode Resets

**WRONG** — Custom camera-controls settings are overwritten when switching
modes:

```typescript
// BAD: Custom settings lost on mode switch
world.camera.controls.truckSpeed = 100;
world.camera.set("Orbit"); // Resets truckSpeed to 2
```

**CORRECT** — Use a custom navigation mode for persistent settings:

```typescript
// GOOD: Custom mode preserves settings across switches
const customMode: OBC.NavigationMode = {
  id: "FastOrbit",
  enabled: false,
  set(active: boolean) {
    this.enabled = active;
    if (active) {
      const controls = world.camera.controls;
      controls.minDistance = 1;
      controls.maxDistance = 300;
      controls.truckSpeed = 100; // Custom speed preserved
    }
  },
};
world.camera.addCustomNavigationMode(customMode);
world.camera.set("FastOrbit");
```
