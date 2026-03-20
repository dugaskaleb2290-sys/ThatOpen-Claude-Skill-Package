# Navigation Examples

## Basic Mode Switching

```typescript
// Default: Orbit mode is active after camera creation
// world.camera.mode.id === "Orbit"

// Switch to FirstPerson (validate projection first)
if (world.camera.projection.current === "Perspective") {
  world.camera.set("FirstPerson");
}

// Switch to Plan mode
world.camera.set("Plan");

// Back to Orbit
world.camera.set("Orbit");
```

## Safe Mode Switching with Projection Validation

```typescript
function setNavigationMode(
  camera: OBC.OrthoPerspectiveCamera,
  mode: OBC.NavModeID,
): void {
  const isOrtho = camera.projection.current === "Orthographic";
  const isFirstPerson = mode === "FirstPerson";

  if (isOrtho && isFirstPerson) {
    console.warn("FirstPerson is not compatible with orthographic projection.");
    return;
  }

  camera.set(mode);
}
```

## Projection Switching

```typescript
// Switch to orthographic
await world.camera.projection.set("Orthographic");

// Switch to perspective
await world.camera.projection.set("Perspective");

// Toggle
await world.camera.projection.toggle();

// Listen for changes
world.camera.projection.onChanged.add(() => {
  const projection = world.camera.projection.current;
  console.log("Now using:", projection);

  // Update grid fade based on projection
  grid.fade = projection === "Perspective";
});
```

## Fit Camera to Objects

```typescript
// Fit to specific meshes
const meshes = [mesh1, mesh2, mesh3];
await world.camera.fit(meshes);

// Fit with more padding
await world.camera.fit(meshes, 2.5);

// Fit to all loaded fragment models
world.camera.fitToItems();
```

## Programmatic Camera Positioning

```typescript
// Set camera position and look-at target
await world.camera.controls.setLookAt(
  68, 23, -8.5,    // camera position (x, y, z)
  21.5, -5.5, 23,  // look-at target (x, y, z)
);

// Animated transition (smooth)
await world.camera.controls.setLookAt(
  68, 23, -8.5,
  21.5, -5.5, 23,
  true, // animate
);
```

## Floor Plan View Setup

```typescript
// Complete floor plan viewing pattern
async function enterFloorPlanView(
  camera: OBC.OrthoPerspectiveCamera,
  grid: OBC.SimpleGrid,
  centerX: number,
  centerZ: number,
  floorHeight: number,
  viewHeight: number,
): Promise<void> {
  // 1. Switch to Plan mode (disables rotation)
  camera.set("Plan");

  // 2. Switch to orthographic projection
  await camera.projection.set("Orthographic");

  // 3. Position camera above floor, looking straight down
  await camera.controls.setLookAt(
    centerX, viewHeight, centerZ,
    centerX, floorHeight, centerZ,
  );

  // 4. Update grid for orthographic
  grid.fade = false;
}

// Exit floor plan view
async function exitFloorPlanView(
  camera: OBC.OrthoPerspectiveCamera,
  grid: OBC.SimpleGrid,
): Promise<void> {
  camera.set("Orbit");
  await camera.projection.set("Perspective");
  grid.fade = true;
}
```

## Custom Navigation Mode

```typescript
// Define a custom "Fly" navigation mode
const flyMode: OBC.NavigationMode = {
  id: "Fly",
  enabled: false,
  set(active: boolean) {
    this.enabled = active;
    if (active) {
      const controls = world.camera.controls;
      controls.minDistance = 0;
      controls.maxDistance = Infinity;
      controls.truckSpeed = 10;
      // No rotation constraints — free flight
    }
  },
};

// Register and activate
world.camera.addCustomNavigationMode(flyMode);
world.camera.set("Fly");

// Switch back to built-in mode
world.camera.set("Orbit");
```

## Lock Camera During Animation

```typescript
// Disable user input during programmatic movement
world.camera.setUserInput(false);

await world.camera.controls.setLookAt(
  newX, newY, newZ,
  targetX, targetY, targetZ,
  true, // animate
);

// Re-enable after animation
world.camera.setUserInput(true);
```

## Camera Update Events with Fragments

```typescript
// Update fragment LOD when camera moves
world.camera.controls.addEventListener("update", () => {
  fragments.core.update();
});

// Update fragment camera reference when world camera changes
world.onCameraChanged.add((camera) => {
  for (const [, model] of fragments.list) {
    model.useCamera(camera.three);
  }
  fragments.core.update(true);
});
```

## UI Dropdown for Mode Selection (ThatOpen UI)

```typescript
import * as BUI from "@thatopen/ui";

BUI.Manager.init();

const panel = BUI.Component.create<BUI.PanelSection>(() => {
  return BUI.html`
    <bim-panel active label="Camera Controls">
      <bim-panel-section label="Navigation">
        <bim-dropdown required label="Navigation Mode"
          @change="${({ target }: { target: BUI.Dropdown }) => {
            const selected = target.value[0] as OBC.NavModeID;
            const isOrtho = world.camera.projection.current === "Orthographic";
            if (isOrtho && selected === "FirstPerson") {
              alert("FirstPerson is not compatible with orthographic!");
              target.value[0] = world.camera.mode.id;
              return;
            }
            world.camera.set(selected);
          }}">
          <bim-option checked label="Orbit"></bim-option>
          <bim-option label="FirstPerson"></bim-option>
          <bim-option label="Plan"></bim-option>
        </bim-dropdown>

        <bim-dropdown required label="Projection"
          @change="${({ target }: { target: BUI.Dropdown }) => {
            const selected = target.value[0] as OBC.CameraProjection;
            const isFirstPerson = world.camera.mode.id === "FirstPerson";
            if (selected === "Orthographic" && isFirstPerson) {
              alert("Orthographic is not compatible with FirstPerson!");
              target.value[0] = world.camera.projection.current;
              return;
            }
            world.camera.projection.set(selected);
          }}">
          <bim-option checked label="Perspective"></bim-option>
          <bim-option label="Orthographic"></bim-option>
        </bim-dropdown>

        <bim-checkbox label="Allow User Input" checked
          @change="${({ target }: { target: BUI.Checkbox }) => {
            world.camera.setUserInput(target.checked);
          }}">
        </bim-checkbox>

        <bim-button label="Fit Model"
          @click=${() => world.camera.fitToItems()}>
        </bim-button>
      </bim-panel-section>
    </bim-panel>
  `;
});
```
