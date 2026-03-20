# OrthoPerspectiveCamera — API Reference

## OrthoPerspectiveCamera

Extends `SimpleCamera`. Provides dual-projection camera with multiple
navigation modes.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `projection` | `ProjectionManager` | Manages perspective/orthographic switching |
| `threePersp` | `THREE.PerspectiveCamera` | Perspective camera instance |
| `threeOrtho` | `THREE.OrthographicCamera` | Orthographic camera instance |
| `three` | `THREE.Camera` | Currently active camera (inherited from SimpleCamera) |
| `controls` | `CameraControls` | Underlying camera-controls instance (inherited) |
| `mode` | `NavigationMode` (getter) | Current active navigation mode |

### Constructor

```typescript
constructor(components: Components)
```

Creates both perspective and orthographic cameras. Initializes the three
built-in navigation modes (Orbit, FirstPerson, Plan). Sets Orbit as the
default active mode.

### Methods

#### set(mode: string): void

Switch the active navigation mode.

```typescript
camera.set("Orbit");
camera.set("FirstPerson");
camera.set("Plan");
camera.set("CustomModeId"); // if registered
```

- Deactivates the current mode by calling `currentMode.set(false)`.
- Activates the new mode by calling `newMode.set(true)`.
- ALWAYS switches modes synchronously.

#### fit(meshes: Iterable\<THREE.Mesh\>, offset?: number): Promise\<void\>

Frame the camera to show all provided meshes.

```typescript
await camera.fit(meshes);          // offset defaults to 1.5
await camera.fit(meshes, 2.0);     // more padding
```

- **meshes**: Iterable of THREE.Mesh objects to frame.
- **offset**: Padding multiplier (default: 1.5). Higher values give more
  space around the objects.

#### fitToItems(): void

Convenience method to fit to all loaded fragment model meshes.

```typescript
camera.fitToItems();
```

#### setUserInput(active: boolean): void

Enable or disable all user camera interactions.

```typescript
camera.setUserInput(false);  // lock camera
camera.setUserInput(true);   // unlock camera
```

When `false`: stores current mouse button assignments, then nullifies them.
When `true`: restores previously stored mouse button assignments.

#### addCustomNavigationMode(mode: NavigationMode): void

Register a custom navigation mode.

```typescript
camera.addCustomNavigationMode(myMode);
```

The mode is stored in the internal `_navigationModes` map keyed by
`mode.id`.

#### dispose(): void

Disposes both cameras, the projection manager, and all navigation modes.
Called automatically when the parent World is disposed.

---

## ProjectionManager

Manages switching between perspective and orthographic projections.

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `current` | `CameraProjection` | `"Perspective"` | Active projection type |
| `onChanged` | `Event<THREE.Camera>` | — | Fires after projection switch |
| `matchOrthoDistanceEnabled` | `boolean` | `false` | Match ortho frustum to perspective distance |

### Methods

#### set(projection: CameraProjection): Promise\<void\>

Switch to a specific projection.

```typescript
await camera.projection.set("Perspective");
await camera.projection.set("Orthographic");
```

ALWAYS await this method. It performs camera transition calculations.

#### toggle(): Promise\<void\>

Toggle between Perspective and Orthographic.

```typescript
await camera.projection.toggle();
```

---

## NavigationMode Interface

```typescript
interface NavigationMode {
  readonly id: NavModeID;    // Unique mode identifier
  enabled: boolean;          // Whether this mode is currently active
  set(active: boolean): void; // Called to activate/deactivate
}
```

When a mode is activated via `set(true)`, it configures camera-controls
settings (rotation speeds, mouse mappings, distance constraints).
When deactivated via `set(false)`, it should restore previous settings.

---

## Type Aliases

```typescript
type NavModeID = "Orbit" | "FirstPerson" | "Plan" | string;
type CameraProjection = "Perspective" | "Orthographic";
```

`NavModeID` is extensible — custom modes can use any string identifier.

---

## Built-in Mode Implementations

### OrbitMode

| Setting | Value |
|---------|-------|
| `minDistance` | 1 |
| `maxDistance` | 300 |
| `truckSpeed` | 2 |

Calculates orbit target from camera direction and distance. Standard
3D CAD-style orbit navigation.

### FirstPersonMode

| Setting | Value |
|---------|-------|
| `minDistance` | 1 |
| `maxDistance` | 1 |
| `distance` | 1 |
| `truckSpeed` | 50 |

Mouse wheel mapped to dolly. Two-finger touch mapped to zoom-truck.
REQUIRES perspective projection — falls back to Orbit if orthographic.

### PlanMode

| Setting | Value |
|---------|-------|
| Azimuth rotation speed | 0 (disabled) |
| Polar rotation speed | 0 (disabled) |
| Left mouse | TRUCK (pan) |
| Single-touch | TOUCH_TRUCK |
| Dual-touch | TOUCH_ZOOM |

Stores original rotation speeds and mouse mappings on activation.
Restores them on deactivation.

---

## CameraControls (Underlying Library)

Key methods available via `camera.controls`:

```typescript
// Position camera
await controls.setLookAt(posX, posY, posZ, targetX, targetY, targetZ, animate?);

// Move target only
await controls.moveTo(x, y, z, animate?);

// Zoom
await controls.dolly(distance, animate?);
await controls.zoom(zoomStep, animate?);

// Listen for updates
controls.addEventListener("update", callback);
controls.removeEventListener("update", callback);
```

See `camera-controls` library documentation for full API.
