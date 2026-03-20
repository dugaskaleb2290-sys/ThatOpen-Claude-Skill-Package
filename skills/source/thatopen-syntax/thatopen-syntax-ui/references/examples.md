# Examples — UI Component Syntax

Version: @thatopen/ui 3.3.x / @thatopen/ui-obc 3.3.x

---

## E-001: Full BIM Application Layout (Grid + Toolbar + Panel + Viewport)

The standard BIM viewer layout uses `bim-grid` to compose toolbar, panel, and
viewport into a responsive CSS Grid.

```typescript
import * as BUI from "@thatopen/ui";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

// ALWAYS init UI before creating any bim-* elements
BUI.Manager.init();

// Engine setup
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();

// Viewport — pass to renderer as container
const viewport = document.createElement("bim-viewport");

world.scene = new OBC.SimpleScene(components);
world.renderer = new OBCF.PostproductionRenderer(components, viewport);
world.camera = new OBC.OrthoPerspectiveCamera(components);
components.init();

// Toolbar
const toolbar = document.createElement("bim-toolbar");

const navSection = document.createElement("bim-toolbar-section");
navSection.label = "Navigation";

const orbitBtn = document.createElement("bim-button");
orbitBtn.label = "Orbit";
orbitBtn.icon = "mdi:orbit";
orbitBtn.addEventListener("click", () => {
  world.camera.set("Orbit");
});

const fitBtn = document.createElement("bim-button");
fitBtn.label = "Fit All";
fitBtn.icon = "mdi:fit-to-screen";
fitBtn.addEventListener("click", () => {
  world.camera.fit(world.meshes);
});

navSection.appendChild(orbitBtn);
navSection.appendChild(fitBtn);
toolbar.appendChild(navSection);

// Panel with sections
const panel = document.createElement("bim-panel");
panel.name = "Properties";
panel.icon = "mdi:information";

const infoSection = document.createElement("bim-panel-section");
infoSection.label = "Model Info";
infoSection.icon = "mdi:cube-outline";

const nameInput = document.createElement("bim-text-input");
nameInput.label = "Project Name";
nameInput.value = "My BIM Project";
infoSection.appendChild(nameInput);

panel.appendChild(infoSection);

// Grid — compose everything
const grid = document.createElement("bim-grid");
grid.layouts = {
  main: {
    template: `
      "toolbar toolbar" auto
      "panel viewport" 1fr
      / 320px 1fr
    `,
    elements: { toolbar, panel, viewport },
  },
};
grid.layout = "main";

document.body.appendChild(grid);
```

---

## E-002: Toolbar with Grouped Actions

Toolbar sections group related buttons. Toolbar groups create nested subgroups.

```typescript
BUI.Manager.init();

const toolbar = document.createElement("bim-toolbar");

// --- Navigation section ---
const navSection = document.createElement("bim-toolbar-section");
navSection.label = "Navigation";

const orbitBtn = document.createElement("bim-button");
orbitBtn.label = "Orbit";
orbitBtn.icon = "mdi:orbit";

const planBtn = document.createElement("bim-button");
planBtn.label = "Plan";
planBtn.icon = "mdi:floor-plan";

navSection.appendChild(orbitBtn);
navSection.appendChild(planBtn);

// --- Tools section with nested group ---
const toolsSection = document.createElement("bim-toolbar-section");
toolsSection.label = "Tools";

const measureGroup = document.createElement("bim-toolbar-group");

const lengthBtn = document.createElement("bim-button");
lengthBtn.label = "Length";
lengthBtn.icon = "mdi:ruler";

const areaBtn = document.createElement("bim-button");
areaBtn.label = "Area";
areaBtn.icon = "mdi:square-outline";

measureGroup.appendChild(lengthBtn);
measureGroup.appendChild(areaBtn);
toolsSection.appendChild(measureGroup);

// --- Assemble ---
toolbar.appendChild(navSection);
toolbar.appendChild(toolsSection);
```

---

## E-003: Property Panel with Form Elements

A panel with multiple sections containing different input types. The panel
aggregates all values via its `value` property.

```typescript
BUI.Manager.init();

const panel = document.createElement("bim-panel");
panel.name = "Element Properties";

// --- General section (always visible) ---
const generalSection = document.createElement("bim-panel-section");
generalSection.label = "General";
generalSection.icon = "mdi:information-outline";
generalSection.fixed = true; // cannot be collapsed

const nameInput = document.createElement("bim-text-input");
nameInput.label = "Name";
nameInput.value = "Wall-001";

const heightInput = document.createElement("bim-number-input");
heightInput.label = "Height";
heightInput.value = 3.0;
heightInput.suffix = "m";
heightInput.min = 0.1;
heightInput.max = 100;
heightInput.step = 0.1;

const structuralCheck = document.createElement("bim-checkbox");
structuralCheck.label = "Structural";
structuralCheck.checked = true;

generalSection.appendChild(nameInput);
generalSection.appendChild(heightInput);
generalSection.appendChild(structuralCheck);

// --- Appearance section (collapsible) ---
const appearanceSection = document.createElement("bim-panel-section");
appearanceSection.label = "Appearance";
appearanceSection.icon = "mdi:palette";
appearanceSection.collapsed = true;

const colorInput = document.createElement("bim-color-input");
colorInput.label = "Color";
colorInput.color = "#ff6600";

const materialDropdown = document.createElement("bim-dropdown");
materialDropdown.label = "Material";

const concrete = document.createElement("bim-option");
concrete.label = "Concrete";
concrete.checked = true;

const steel = document.createElement("bim-option");
steel.label = "Steel";

const wood = document.createElement("bim-option");
wood.label = "Wood";

materialDropdown.appendChild(concrete);
materialDropdown.appendChild(steel);
materialDropdown.appendChild(wood);

appearanceSection.appendChild(colorInput);
appearanceSection.appendChild(materialDropdown);

// --- Assemble and listen ---
panel.appendChild(generalSection);
panel.appendChild(appearanceSection);

panel.addEventListener("change", () => {
  console.log("Panel values:", panel.value);
  // { Name: "Wall-001", Height: 3.0, Structural: true,
  //   Color: "#ff6600", Material: "Concrete" }
});
```

---

## E-004: Data Table with Hierarchy and Selection

Displaying IFC spatial structure data in a `bim-table` with expandable
hierarchy and row selection.

```typescript
BUI.Manager.init();

const table = document.createElement("bim-table");

// Define columns with custom widths
table.columns = [
  { name: "Name", width: "minmax(200px, 1fr)" },
  { name: "Type", width: "150px" },
  { name: "Level", width: "100px" },
];

// Hierarchical data
table.data = [
  {
    data: { Name: "Building A", Type: "IfcBuilding", Level: "-" },
    children: [
      {
        data: { Name: "Level 1", Type: "IfcBuildingStorey", Level: "0.000" },
        children: [
          { data: { Name: "Wall-001", Type: "IfcWall", Level: "0.000" } },
          { data: { Name: "Wall-002", Type: "IfcWall", Level: "0.000" } },
          { data: { Name: "Slab-001", Type: "IfcSlab", Level: "0.000" } },
        ],
      },
      {
        data: { Name: "Level 2", Type: "IfcBuildingStorey", Level: "3.500" },
        children: [
          { data: { Name: "Wall-003", Type: "IfcWall", Level: "3.500" } },
        ],
      },
    ],
  },
];

// Enable selection
table.selectableRows = true;
table.expanded = true;

// Listen for selection
table.addEventListener("dataselected", (e) => {
  const row = e.detail.data;
  console.log("Selected:", row.Name, row.Type);
});

table.addEventListener("datadeselected", (e) => {
  console.log("Deselected:", e.detail.data.Name);
});
```

### Filtering the Table

```typescript
// Simple text filter — searches all columns
table.queryString = "wall";

// Column-specific query
table.queryString = 'Type="IfcWall" & Level="0.000"';

// Custom filter function
table.filterFunction = (row) => {
  return row.data.Type === "IfcWall";
};

// Group by column
table.groupedBy = "Level";

// Export to CSV
table.downloadData("elements", "csv");
```

---

## E-005: Using @thatopen/ui-obc Functional Components

Pre-built BIM UI components that wire directly to the ThatOpen engine.

```typescript
import * as BUI from "@thatopen/ui";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as CUI from "@thatopen/ui-obc";

BUI.Manager.init();

const components = new OBC.Components();
// ... world setup omitted for brevity ...

const fragments = components.get(OBC.FragmentsManager);

// --- IFC Load Button ---
const [loadIfcBtn, updateLoadIfcBtn] = CUI.buttons.loadIfc({ components });

// --- Spatial Tree ---
const [spatialTree, updateSpatialTree] = CUI.tables.spatialTree({
  components,
  models: [],
});

// Update the tree when a model is loaded
fragments.onFragmentsLoaded.add((model) => {
  updateSpatialTree({ models: [model] });
});

// --- Models List ---
const [modelsList, updateModelsList] = CUI.tables.modelsList({ components });

// --- Compose into panel ---
const panel = document.createElement("bim-panel");
panel.name = "Model Browser";

const treeSection = document.createElement("bim-panel-section");
treeSection.label = "Spatial Structure";
treeSection.appendChild(spatialTree);

const modelsSection = document.createElement("bim-panel-section");
modelsSection.label = "Models";
modelsSection.appendChild(modelsList);

panel.appendChild(treeSection);
panel.appendChild(modelsSection);

// --- Add load button to toolbar ---
const toolbar = document.createElement("bim-toolbar");
const fileSection = document.createElement("bim-toolbar-section");
fileSection.label = "File";
fileSection.appendChild(loadIfcBtn);
toolbar.appendChild(fileSection);
```

---

## E-006: CSS Theming — Custom Dark Theme

Override CSS custom properties to create a branded dark theme.

```css
/* Apply to root or any container element */
:root {
  --bim-ui_bg-base: #0d1117;
  --bim-ui_bg-contrast-10: #161b22;
  --bim-ui_bg-contrast-20: #21262d;
  --bim-ui_bg-contrast-40: #30363d;
  --bim-ui_bg-contrast-60: #484f58;
  --bim-ui_bg-contrast-80: #8b949e;
  --bim-ui_bg-contrast-100: #c9d1d9;

  --bim-ui_accent-base: #238636;
  --bim-ui_accent-contrast: #ffffff;

  --bim-ui_font-family: "JetBrains Mono", monospace;
  --bim-ui_size-base: 0.3rem;
}
```

### Scoped Theming

Override properties on a specific container to theme only part of the UI:

```css
/* Only this panel gets custom colors */
#settings-panel {
  --bim-ui_bg-base: #1e293b;
  --bim-ui_accent-base: #3b82f6;
}
```

---

## E-007: Panel Toggle with Activation Button

Every `bim-panel` creates an `activationButton` that toggles visibility.

```typescript
BUI.Manager.init();

const panel = document.createElement("bim-panel");
panel.name = "Properties";
panel.hidden = true; // start hidden

// Get the auto-created toggle button
const toggleBtn = panel.activationButton;
toggleBtn.label = "Properties";
toggleBtn.icon = "mdi:dock-right";

// Place in toolbar
const toolbar = document.createElement("bim-toolbar");
const viewSection = document.createElement("bim-toolbar-section");
viewSection.label = "View";
viewSection.appendChild(toggleBtn);
toolbar.appendChild(viewSection);

// Listen for visibility changes
panel.addEventListener("hiddenchange", () => {
  console.log("Panel visible:", !panel.hidden);
});
```

---

## E-008: Layout Switching with Grid

Switch between different layouts at runtime (e.g., full viewport vs. split view).

```typescript
BUI.Manager.init();

const grid = document.createElement("bim-grid");

grid.layouts = {
  split: {
    template: `
      "toolbar toolbar" auto
      "panel viewport" 1fr
      / 320px 1fr
    `,
    elements: { toolbar, panel, viewport },
  },
  fullscreen: {
    template: `
      "toolbar" auto
      "viewport" 1fr
      / 1fr
    `,
    elements: { toolbar, viewport },
  },
};

grid.layout = "split";

// Toggle layout button
const toggleLayoutBtn = document.createElement("bim-button");
toggleLayoutBtn.label = "Toggle Layout";
toggleLayoutBtn.icon = "mdi:fullscreen";
toggleLayoutBtn.addEventListener("click", () => {
  grid.layout = grid.layout === "split" ? "fullscreen" : "split";
});

// Listen for layout changes
grid.addEventListener("layoutchange", () => {
  console.log("Current layout:", grid.layout);
});
```

---

## E-009: Tabs for Multiple Panels

Use `bim-tabs` to organize multiple views within a single panel area.

```html
<bim-panel name="Inspector">
  <bim-tabs>
    <bim-tab label="Properties" icon="mdi:format-list-bulleted">
      <bim-panel-section label="Attributes" icon="mdi:tag">
        <bim-text-input label="Name" value="Wall-001"></bim-text-input>
        <bim-text-input label="GlobalId" value="2O2Fr$t4X7Z..."></bim-text-input>
      </bim-panel-section>
    </bim-tab>
    <bim-tab label="Relations" icon="mdi:link-variant">
      <bim-panel-section label="Contains" icon="mdi:folder-open">
        <!-- relation content -->
      </bim-panel-section>
    </bim-tab>
  </bim-tabs>
</bim-panel>
```

---

## E-010: Resizable Grid Areas

Enable the user to drag-resize grid areas at runtime.

```typescript
BUI.Manager.init();

const grid = document.createElement("bim-grid");
grid.resizeableAreas = true;

// Prevent the toolbar area from being resized
grid.areasResizeExceptions = ["toolbar"];

grid.layouts = {
  main: {
    template: `
      "toolbar toolbar" auto
      "panel viewport" 1fr
      / 320px 1fr
    `,
    elements: { toolbar, panel, viewport },
  },
};
grid.layout = "main";

// The user can now drag the border between panel and viewport.
// The toolbar row is excluded from resize interactions.
```
