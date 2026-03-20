# Measurement Examples

## Prerequisites

All examples assume a working ThatOpen viewer (see thatopen-impl-viewer).
The world, components, and container are already set up.

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

// Viewer already initialized (see thatopen-impl-viewer)
const components = new OBC.Components();
// ... world setup ...
components.init();
```

---

## LengthMeasurement — Free Mode

Measure the distance between two arbitrary points.

```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.units = "m";
lengths.rounding = 2;
lengths.snapDistance = 0.25;
lengths.mode = "free";
lengths.enabled = true;

// User clicks two points in the 3D viewport.
// A DimensionLine with distance label appears automatically.

// To programmatically end or cancel:
// lengths.endCreation();    // finalize current measurement
// lengths.cancelCreation(); // discard current measurement
```

## LengthMeasurement — Edge Mode

Snap to geometry edges and measure their length.

```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.mode = "edge";
lengths.enabled = true;

// User hovers over edges — snap marker appears on edge.
// Click to measure the full edge length.
```

## LengthMeasurement — Custom Color and Endpoint

```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.color = new THREE.Color(0xff0000); // red measurements

// Custom endpoint element
const endpoint = document.createElement("div");
endpoint.style.backgroundColor = "red";
endpoint.style.width = "10px";
endpoint.style.height = "10px";
endpoint.style.borderRadius = "50%";
lengths.linesEndpointElement = endpoint;

lengths.mode = "free";
lengths.enabled = true;
```

---

## AreaMeasurement — Free Mode

Define a polygon area by clicking 3+ points.

```typescript
const areas = components.get(OBCF.AreaMeasurement);
areas.world = world;
areas.units = "m2";
areas.rounding = 2;
areas.mode = "free";
areas.enabled = true;

// User clicks points to define polygon vertices.
// After 3+ points, call endCreation() to finalize:
// areas.endCreation();

// A MeasureFill with perimeter lines and area label appears.
```

## AreaMeasurement — Square Mode

Define a rectangular area with two clicks.

```typescript
const areas = components.get(OBCF.AreaMeasurement);
areas.world = world;
areas.mode = "square";
areas.enabled = true;

// User clicks two opposite corners of the rectangle.
// The rectangular area is computed automatically.
```

## AreaMeasurement — Face Mode

Click a geometry face to measure its area directly.

```typescript
const areas = components.get(OBCF.AreaMeasurement);
areas.world = world;
areas.mode = "face";
areas.pickTolerance = 0.1;
areas.tolerance = 0.005;
areas.enabled = true;

// User clicks a face in the model.
// The face area is calculated and displayed.
```

---

## VolumeMeasurement — Free Mode

Define a volume by selecting geometry faces.

```typescript
const volumes = components.get(OBCF.VolumeMeasurement);
volumes.world = world;
volumes.units = "m3";
volumes.rounding = 3;
volumes.mode = "free";

// Listen for preview initialization
volumes.onPreviewInitialized.add((preview) => {
  console.log("Volume preview ready:", preview);
});

volumes.enabled = true;

// User clicks faces to define volume boundary.
// Call endCreation() to finalize:
// volumes.endCreation();
```

---

## AngleMeasurement — Free Mode

Measure the angle between three points. The angle is measured at the
second (middle) point.

```typescript
const angles = components.get(OBCF.AngleMeasurement);
angles.world = world;
angles.units = "deg";
angles.rounding = 1;
angles.mode = "free";
angles.enabled = true;

// User clicks three points:
//   1. First arm endpoint
//   2. Vertex (angle center)
//   3. Second arm endpoint
// An arc with angle label appears at the vertex.
```

---

## Custom Value Formatter

Apply a custom format to ALL measurement labels globally.

```typescript
import { Measurement } from "@thatopen/components-front";

// Set BEFORE enabling measurements
Measurement.valueFormatter = (value: number) => {
  if (value < 1) {
    return `${(value * 100).toFixed(0)} cm`;
  }
  return `${value.toFixed(2)} m`;
};

// Now enable measurements — all labels use the custom formatter
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.enabled = true;
```

---

## Switching Between Measurement Types

ALWAYS disable the current type before enabling another.

```typescript
const lengths = components.get(OBCF.LengthMeasurement);
const areas = components.get(OBCF.AreaMeasurement);
const angles = components.get(OBCF.AngleMeasurement);

// Configure all types with world reference
lengths.world = world;
areas.world = world;
angles.world = world;

// Start with length measurement
lengths.enabled = true;

function switchToArea() {
  lengths.enabled = false;
  areas.mode = "free";
  areas.enabled = true;
}

function switchToAngle() {
  areas.enabled = false;
  angles.enabled = true;
}

function disableAll() {
  lengths.enabled = false;
  areas.enabled = false;
  angles.enabled = false;
}
```

---

## Deleting Measurements

### Interactive Deletion (Under Cursor)

```typescript
// Delete the length measurement under the cursor
lengths.delete();

// Delete the area measurement under the cursor
areas.delete();
```

### Clear All Measurements of a Type

```typescript
// Clear all length measurements
lengths.list.clear();

// Clear all area measurements
areas.list.clear();
```

---

## Listening to State Changes

```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;

lengths.onStateChanged.add((changes) => {
  console.log("State changed:", changes);
  // changes is an array like ["mode"], ["color"], ["units", "rounding"]
});

lengths.onEnabledChange.add((enabled) => {
  console.log("Measurements enabled:", enabled);
});

lengths.onVisibilityChange.add((visible) => {
  console.log("Measurements visible:", visible);
});
```

---

## Snapping Configuration

```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;

// Increase snap distance for large models
lengths.snapDistance = 1.0;

// Increase picker marker size for visibility
lengths.pickerSize = 10;

// The snappings array controls which snap types are active.
// Default includes LINE, POINT, and FACE.
// Access via the snappings property on the measurement instance.

lengths.enabled = true;
```

---

## Unit Conversion

Change units after measurements are created — values update automatically.

```typescript
const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.units = "m";
lengths.enabled = true;

// ... user creates measurements ...

// Switch to centimeters — all existing labels update
lengths.units = "cm";

// Switch to millimeters
lengths.units = "mm";
```

---

## Clipping Plane Integration

Make measurements respect clipping planes.

```typescript
const clipper = components.get(OBC.Clipper);
// ... create clipping planes ...

const lengths = components.get(OBCF.LengthMeasurement);
lengths.world = world;
lengths.enabled = true;

// Apply clipping plane visibility to measurements
const planes = [new THREE.Plane(new THREE.Vector3(0, -1, 0), 5)];
lengths.applyPlanesVisibility(planes);
```

---

## Full Measurement Toolbar Example

Complete setup with all measurement types and a toolbar for switching.

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

// Assume viewer is set up (world, components, container)

// Initialize all measurement types
const lengths = components.get(OBCF.LengthMeasurement);
const areas = components.get(OBCF.AreaMeasurement);
const volumes = components.get(OBCF.VolumeMeasurement);
const angles = components.get(OBCF.AngleMeasurement);

// Configure all with world reference
[lengths, areas, volumes, angles].forEach((m) => {
  m.world = world;
  m.rounding = 2;
  m.snapDistance = 0.25;
});

// Custom formatter for all types
Measurement.valueFormatter = (value: number) => value.toFixed(2);

// Active measurement tracker
let activeMeasurement: typeof lengths | typeof areas | typeof volumes | typeof angles | null = null;

function activate(measurement: typeof activeMeasurement) {
  if (activeMeasurement) {
    activeMeasurement.enabled = false;
  }
  activeMeasurement = measurement;
  if (measurement) {
    measurement.enabled = true;
  }
}

// Toolbar handlers
document.getElementById("btn-length")?.addEventListener("click", () => {
  lengths.mode = "free";
  activate(lengths);
});

document.getElementById("btn-area")?.addEventListener("click", () => {
  areas.mode = "free";
  activate(areas);
});

document.getElementById("btn-volume")?.addEventListener("click", () => {
  volumes.mode = "free";
  activate(volumes);
});

document.getElementById("btn-angle")?.addEventListener("click", () => {
  angles.mode = "free";
  activate(angles);
});

document.getElementById("btn-clear")?.addEventListener("click", () => {
  activate(null);
});

// Cleanup
window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

---

## Disposal Pattern

ALWAYS dispose measurements when the viewer is destroyed.

```typescript
// Option 1: Dispose individual measurement types
lengths.dispose();
areas.dispose();
volumes.dispose();
angles.dispose();

// Option 2: Let components.dispose() handle everything (preferred)
components.dispose();
```

After disposal, NEVER access any measurement properties or methods.
