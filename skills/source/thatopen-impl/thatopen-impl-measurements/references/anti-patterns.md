# Measurement Anti-Patterns

## AP-1: Missing World Assignment

**Wrong**:
```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.enabled = true; // No world assigned!
// Result: vertex picker has no world reference, raycasting fails silently.
```

**Correct**:
```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world; // ALWAYS assign world first
lengths.enabled = true;
```

ALWAYS assign `world` before setting `enabled = true`. The vertex picker
requires a valid world reference to perform raycasting.

---

## AP-2: Multiple Measurement Types Enabled Simultaneously

**Wrong**:
```typescript
lengths.enabled = true;
areas.enabled = true; // Both enabled — pointer events conflict!
// Result: unpredictable behavior, clicks register on both measurement types.
```

**Correct**:
```typescript
lengths.enabled = true;

// When switching to areas:
lengths.enabled = false; // ALWAYS disable first
areas.enabled = true;
```

NEVER have multiple measurement types enabled at the same time. They share
pointer events and will interfere with each other.

---

## AP-3: Missing Dispose on Cleanup

**Wrong**:
```typescript
// Component unmounts, viewer destroyed
// Measurement instances are just abandoned — no dispose called.
// Result: memory leak, GPU resources not freed, event listeners remain active.
```

**Correct**:
```typescript
// Option 1: Dispose individually
lengths.dispose();
areas.dispose();

// Option 2: Let components.dispose() handle all (preferred)
components.dispose();
```

ALWAYS call `dispose()` when measurements are no longer needed. Each
measurement instance holds DimensionLine elements with Three.js geometries,
materials, and HTML overlays that MUST be cleaned up.

---

## AP-4: Missing endCreation() Call

**Wrong**:
```typescript
lengths.enabled = true;
// User clicks two points, measurement preview appears
lengths.enabled = false; // Disabled without finalizing!
// Result: preview elements leak, measurement not saved to list.
```

**Correct**:
```typescript
lengths.enabled = true;
// User clicks two points
lengths.endCreation(); // ALWAYS finalize before disabling
lengths.enabled = false;
```

For programmatic workflows, ALWAYS call `endCreation()` to save the
measurement or `cancelCreation()` to discard it before disabling.

---

## AP-5: Missing cancelCreation() on Abort

**Wrong**:
```typescript
// User started measuring but wants to cancel
lengths.enabled = false; // Just disabling — preview elements leak!
lengths.enabled = true;  // Re-enable — orphaned preview still exists.
```

**Correct**:
```typescript
// User wants to cancel
lengths.cancelCreation(); // ALWAYS cancel before disabling
lengths.enabled = false;
```

ALWAYS call `cancelCreation()` when discarding an in-progress measurement.
Simply disabling the measurement type does not clean up preview elements.

---

## AP-6: Setting valueFormatter After Measurements Exist

**Wrong**:
```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.enabled = true;
// ... user creates several measurements ...

// Now setting formatter — existing labels do NOT update
Measurement.valueFormatter = (v) => `${v.toFixed(1)} m`;
```

**Correct**:
```typescript
// Set formatter BEFORE enabling measurements
Measurement.valueFormatter = (v) => `${v.toFixed(1)} m`;

const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.enabled = true;
```

ALWAYS set `Measurement.valueFormatter` before enabling measurements and
creating the first measurement. Existing labels do not retroactively update
when the formatter changes.

---

## AP-7: Wrong Snap Distance for Model Scale

**Wrong**:
```typescript
// Model is in millimeters (building is ~50000 mm wide)
lengths.snapDistance = 0.25; // Default — way too small for mm-scale models
// Result: snapping never activates, user cannot snap to vertices/edges.
```

**Correct**:
```typescript
// For mm-scale models, increase snap distance
lengths.snapDistance = 50; // Appropriate for millimeter-scale geometry

// For meter-scale models, default is usually fine
lengths.snapDistance = 0.25;
```

ALWAYS configure `snapDistance` relative to your model's coordinate scale.
The default value assumes meter-scale geometry. For models in millimeters
or other units, adjust proportionally.

---

## AP-8: Using Measurement After Dispose

**Wrong**:
```typescript
lengths.dispose();
lengths.enabled = true; // Using after dispose!
// Result: errors, undefined behavior, potential crashes.
```

**Correct**:
```typescript
lengths.dispose();
// NEVER access the instance after dispose.
// If you need measurements again, get a new instance from components.
```

NEVER use a measurement instance after calling `dispose()`. The DataSets,
materials, and vertex picker are all destroyed.

---

## AP-9: Not Handling Mode Before Enabling

**Wrong**:
```typescript
const areas = components.get(OBCF.AreaMeasurement);
areas.world = world;
areas.enabled = true; // Using default mode — might not be what you want
```

**Correct**:
```typescript
const areas = components.get(OBCF.AreaMeasurement);
areas.world = world;
areas.mode = "face"; // Explicitly set desired mode
areas.enabled = true;
```

ALWAYS set the `mode` property explicitly before enabling. The default mode
may not match the desired interaction behavior.

---

## AP-10: Ignoring Minimum Points for Area Measurement

**Wrong**:
```typescript
areas.mode = "free";
areas.enabled = true;
// User clicks only 2 points
areas.endCreation(); // Not enough points!
// Result: endCreation() does nothing — minimum 3 points required.
```

**Correct**:
```typescript
areas.mode = "free";
areas.enabled = true;
// User clicks 3+ points to define a closed polygon
areas.endCreation(); // Finalizes with valid polygon
```

In "free" mode, AreaMeasurement requires a minimum of 3 points to create a
valid polygon. ALWAYS ensure enough points are placed before calling
`endCreation()`.

---

## AP-11: Forgetting to Exclude Grid from PostproductionRenderer

**Wrong**:
```typescript
// PostproductionRenderer is active
lengths.enabled = true;
// Measurement lines render with post-processing artifacts.
```

This is not a measurement-specific issue but affects measurement visibility.
See thatopen-impl-viewer for the grid exclusion pattern. Measurement visual
elements may also need consideration with post-processing.

---

## AP-12: Not Listening for State Changes in UI

**Wrong**:
```typescript
// UI shows "Length Mode: free" but user changed mode programmatically
lengths.mode = "edge";
// UI is now out of sync — no listener registered.
```

**Correct**:
```typescript
lengths.onStateChanged.add((changes) => {
  if (changes.includes("mode")) {
    updateModeIndicator(lengths.mode);
  }
  if (changes.includes("units")) {
    updateUnitsDisplay(lengths.units);
  }
});
```

ALWAYS subscribe to `onStateChanged` when building UI that reflects
measurement state. This ensures the UI stays synchronized with programmatic
or user-driven state changes.

---

## Summary Table

| # | Anti-Pattern | Consequence | Fix |
|---|---|---|---|
| AP-1 | Missing world assignment | Silent raycasting failure | Assign world before enable |
| AP-2 | Multiple types enabled | Pointer event conflicts | Disable before switching |
| AP-3 | Missing dispose | Memory/GPU leak | Call dispose on cleanup |
| AP-4 | Missing endCreation | Preview elements leak | Finalize or cancel |
| AP-5 | Missing cancelCreation | Orphaned preview elements | Cancel before disable |
| AP-6 | Late valueFormatter | Existing labels unchanged | Set formatter before enable |
| AP-7 | Wrong snap distance | Snapping fails | Scale to model units |
| AP-8 | Use after dispose | Errors/crashes | Never use after dispose |
| AP-9 | Default mode assumed | Wrong interaction | Set mode explicitly |
| AP-10 | Too few area points | endCreation no-op | Ensure 3+ points |
| AP-11 | Grid not excluded | Visual artifacts | Exclude grid from PP |
| AP-12 | No state listeners | UI out of sync | Subscribe to onStateChanged |
