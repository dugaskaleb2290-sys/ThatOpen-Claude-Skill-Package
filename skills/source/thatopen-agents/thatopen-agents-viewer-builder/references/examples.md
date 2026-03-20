# Viewer Builder — Complete Working Application

This file contains a complete, copy-pasteable ThatOpen BIM viewer
application. All files below form a working Vite + TypeScript project.

---

## File: package.json

```json
{
  "name": "thatopen-bim-viewer",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@thatopen/components": "^3.3.3",
    "@thatopen/components-front": "^3.3.3",
    "@thatopen/fragments": "^3.3.6",
    "@thatopen/ui": "^3.3.3",
    "@thatopen/ui-obc": "^3.3.3",
    "three": "^0.175.0",
    "web-ifc": "^0.0.77"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "vite": "^5.4.0"
  }
}
```

---

## File: vite.config.ts

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    {
      name: "coop-coep-headers",
      configureServer(server) {
        server.middlewares.use((_req, res, next) => {
          res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
          res.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
          next();
        });
      },
    },
  ],
  optimizeDeps: {
    exclude: ["web-ifc"],
  },
  worker: {
    format: "es",
  },
});
```

---

## File: index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>BIM Viewer</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    html, body {
      width: 100%;
      height: 100%;
      overflow: hidden;
      font-family: sans-serif;
    }
  </style>
</head>
<body>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

---

## File: src/main.ts

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as BUI from "@thatopen/ui";

// ============================================================
// 1. INITIALIZE UI
// ============================================================

BUI.Manager.init();

// ============================================================
// 2. CREATE COMPONENTS CONTAINER
// ============================================================

const components = new OBC.Components();

// ============================================================
// 3. CREATE WORLD (scene, renderer, camera)
// ============================================================

const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

// Viewport element — the renderer binds to this
const viewport = document.createElement("bim-viewport");

// Scene with default lights
world.scene = new OBC.SimpleScene(components);
world.scene.setup();

// Renderer with post-processing
world.renderer = new OBCF.PostproductionRenderer(components, viewport);
world.renderer.postproduction.enabled = true;

// Camera with orbit controls
world.camera = new OBC.OrthoPerspectiveCamera(components);

// ============================================================
// 4. GRID (excluded from post-processing)
// ============================================================

const grids = components.get(OBC.Grids);
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);

// ============================================================
// 5. IFC LOADER SETUP
// ============================================================

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup(); // autoSetWasm: true by default

// ============================================================
// 6. HIGHLIGHTER (click-to-select)
// ============================================================

const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

// ============================================================
// 7. START RENDER LOOP
// ============================================================

components.init();

// ============================================================
// 8. BUILD TOOLBAR
// ============================================================

// --- Load IFC button ---
const loadIfcBtn = document.createElement("bim-button");
loadIfcBtn.label = "Load IFC";
loadIfcBtn.icon = "mdi:file-upload";

const fileInput = document.createElement("input");
fileInput.type = "file";
fileInput.accept = ".ifc";
fileInput.style.display = "none";
document.body.appendChild(fileInput);

loadIfcBtn.addEventListener("click", () => {
  fileInput.click();
});

fileInput.addEventListener("change", async () => {
  const file = fileInput.files?.[0];
  if (!file) return;

  loadIfcBtn.label = "Loading...";
  loadIfcBtn.disabled = true;

  try {
    const buffer = await file.arrayBuffer();
    const data = new Uint8Array(buffer);
    await ifcLoader.load(data, true, file.name);
    world.camera.fit(world.meshes);
  } catch (error) {
    console.error("IFC loading failed:", error);
  } finally {
    loadIfcBtn.label = "Load IFC";
    loadIfcBtn.disabled = false;
    fileInput.value = ""; // allow re-selecting same file
  }
});

// --- Fit view button ---
const fitBtn = document.createElement("bim-button");
fitBtn.label = "Fit View";
fitBtn.icon = "mdi:fit-to-screen";
fitBtn.addEventListener("click", () => {
  world.camera.fit(world.meshes);
});

// --- Clear selection button ---
const clearBtn = document.createElement("bim-button");
clearBtn.label = "Clear Selection";
clearBtn.icon = "mdi:select-off";
clearBtn.addEventListener("click", () => {
  highlighter.clear();
});

// --- Toolbar assembly ---
const toolbarSection = document.createElement("bim-toolbar-section");
toolbarSection.label = "Tools";
toolbarSection.appendChild(loadIfcBtn);
toolbarSection.appendChild(fitBtn);
toolbarSection.appendChild(clearBtn);

const toolbar = document.createElement("bim-toolbar");
toolbar.appendChild(toolbarSection);

// ============================================================
// 9. BUILD PANEL
// ============================================================

const panel = document.createElement("bim-panel");
panel.name = "info";
panel.label = "Model Info";
panel.icon = "mdi:information";

const infoSection = document.createElement("bim-panel-section");
infoSection.label = "Instructions";
infoSection.fixed = true;

const infoLabel = document.createElement("bim-label");
infoLabel.textContent = "Click 'Load IFC' to open an IFC file. Click elements in the viewport to select them.";
infoSection.appendChild(infoLabel);

const selectionSection = document.createElement("bim-panel-section");
selectionSection.label = "Selection";

const selectionLabel = document.createElement("bim-label");
selectionLabel.textContent = "No element selected";
selectionSection.appendChild(selectionLabel);

// Update selection info when highlight changes
highlighter.events.select.onHighlight.add((data) => {
  const modelIds = Object.values(data);
  let totalItems = 0;
  for (const idMap of modelIds) {
    for (const [, ids] of idMap) {
      totalItems += ids.size;
    }
  }
  selectionLabel.textContent = `${totalItems} element(s) selected`;
});

highlighter.events.select.onClear.add(() => {
  selectionLabel.textContent = "No element selected";
});

panel.appendChild(infoSection);
panel.appendChild(selectionSection);

// ============================================================
// 10. ASSEMBLE LAYOUT
// ============================================================

const grid2 = document.createElement("bim-grid");
grid2.layouts = {
  main: {
    template: `
      "toolbar toolbar" auto
      "panel viewport" 1fr
      / 320px 1fr
    `,
    elements: {
      toolbar,
      panel,
      viewport,
    },
  },
};
grid2.layout = "main";
document.body.appendChild(grid2);

// ============================================================
// 11. DISPOSAL
// ============================================================

window.addEventListener("beforeunload", () => {
  components.dispose();
});
```

---

## File: tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true
  },
  "include": ["src"]
}
```

---

## Quick Start Commands

```bash
# 1. Create project
npm create vite@latest my-bim-viewer -- --template vanilla-ts
cd my-bim-viewer

# 2. Install dependencies
npm install @thatopen/components@^3.3.3 @thatopen/components-front@^3.3.3 \
  @thatopen/fragments@^3.3.6 @thatopen/ui@^3.3.3 @thatopen/ui-obc@^3.3.3 \
  three@^0.175.0 web-ifc@^0.0.77

# 3. Replace files with the above content
# (vite.config.ts, index.html, src/main.ts, tsconfig.json)

# 4. Clean up boilerplate
rm -f src/counter.ts src/style.css src/typescript.svg public/vite.svg

# 5. Run
npm run dev
```

---

## Minimal Viewer (Without UI Components)

If the user only needs a basic viewer without the `@thatopen/ui` panel
and toolbar, use this stripped-down version:

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

// Container — must have explicit dimensions
const container = document.getElementById("viewer") as HTMLDivElement;
if (!container) throw new Error("Container #viewer not found");

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.OrthoPerspectiveCamera,
  OBCF.PostproductionRenderer
>();

world.scene = new OBC.SimpleScene(components);
world.scene.setup();
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.renderer.postproduction.enabled = true;
world.camera = new OBC.OrthoPerspectiveCamera(components);

const grids = components.get(OBC.Grids);
const grid = grids.create(world);
world.renderer.postproduction.exclude.add(grid.three);

const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });

components.init();

// Load an IFC file
async function loadIfc(url: string) {
  const response = await fetch(url);
  const data = new Uint8Array(await response.arrayBuffer());
  await ifcLoader.load(data, true, "model");
  world.camera.fit(world.meshes);
}

window.addEventListener("beforeunload", () => components.dispose());
```

HTML for the minimal version:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <style>
    * { margin: 0; padding: 0; }
    #viewer { width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <div id="viewer"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```
