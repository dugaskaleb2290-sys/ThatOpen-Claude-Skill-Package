# Viewer Builder — Setup Checklist & Configuration Reference

## Complete Setup Checklist

Use this checklist when scaffolding a new ThatOpen BIM viewer. Every item
is REQUIRED unless marked optional.

### Project Initialization

- [ ] `npm create vite@latest <name> -- --template vanilla-ts`
- [ ] Install runtime dependencies (see Package Versions below)
- [ ] Install dev dependencies: `typescript@^5.4.0`, `vite@^5.4.0`
- [ ] Create `vite.config.ts` with COOP/COEP plugin, optimizeDeps, worker format
- [ ] Replace `index.html` with full-viewport template
- [ ] Replace `src/main.ts` with viewer code
- [ ] Delete boilerplate: `counter.ts`, `style.css`, SVG files

### Code Initialization Order

- [ ] `BUI.Manager.init()` — register UI custom elements
- [ ] `new OBC.Components()` — create component container
- [ ] `components.get(OBC.Worlds).create()` — create world
- [ ] `world.scene = new OBC.SimpleScene(components)` — assign scene
- [ ] `world.scene.setup()` — initialize default lights
- [ ] `world.renderer = new OBCF.PostproductionRenderer(components, container)` — assign renderer
- [ ] `world.renderer.postproduction.enabled = true` — enable post-processing
- [ ] `world.camera = new OBC.OrthoPerspectiveCamera(components)` — assign camera
- [ ] `components.get(OBC.Grids).create(world)` — create grid
- [ ] Exclude grid from postproduction — `world.renderer.postproduction.exclude.add(grid.three)`
- [ ] `components.get(OBC.FragmentsManager).init(workerURL)` — (optional, for worker-based loading)
- [ ] `components.get(OBC.IfcLoader).setup()` — configure WASM
- [ ] `components.get(OBCF.Highlighter).setup({ world })` — enable selection
- [ ] `components.init()` — start render loop
- [ ] Build UI (toolbar, panel, grid layout)
- [ ] Append `bim-grid` to `document.body`
- [ ] Register disposal on `beforeunload`

### Verification

- [ ] `npm run dev` — starts without errors
- [ ] 3D viewport with grid visible in browser
- [ ] Orbit controls respond to mouse interaction
- [ ] IFC file loads via file input
- [ ] Clicking an element highlights it
- [ ] No console errors related to WASM or SharedArrayBuffer

---

## Package Versions

| Package | Install Version | Peer Requirements |
|---|---|---|
| `@thatopen/components` | `^3.3.3` | fragments ~3.3.0, three >=0.175, web-ifc >=0.0.74 |
| `@thatopen/components-front` | `^3.3.3` | fragments ~3.3.0, three >=0.175, web-ifc >=0.0.74 |
| `@thatopen/fragments` | `^3.3.6` | three >=0.175, web-ifc >=0.0.74 |
| `@thatopen/ui` | `^3.3.3` | standalone |
| `@thatopen/ui-obc` | `^3.3.3` | components ~3.3.0, components-front ~3.3.0, fragments ~3.3.0, three >=0.175 |
| `three` | `^0.175.0` | standalone |
| `web-ifc` | `^0.0.77` | standalone (WASM) |
| `typescript` | `^5.4.0` | dev only |
| `vite` | `^5.4.0` | dev only |

---

## Vite Configuration Reference

### COOP/COEP Headers Plugin

```typescript
{
  name: "coop-coep-headers",
  configureServer(server) {
    server.middlewares.use((_req, res, next) => {
      res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
      res.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
      next();
    });
  },
}
```

Purpose: Enable `SharedArrayBuffer` for web-ifc WASM multithreading.
Without these headers, the browser disables `SharedArrayBuffer` and WASM
operations fail.

For production builds, configure these headers on your web server (nginx,
Apache, Cloudflare, etc.) — the Vite plugin only works in dev mode.

### optimizeDeps Configuration

```typescript
optimizeDeps: {
  exclude: ["web-ifc"],
}
```

Purpose: Prevent Vite from pre-bundling web-ifc. The WASM module loader
breaks when Vite transforms it.

### Worker Configuration

```typescript
worker: {
  format: "es",
}
```

Purpose: Fragment processing workers use ES module format. Without this,
worker imports fail in development.

---

## Component Configuration Reference

### PostproductionRenderer

| Property | Type | Default | Description |
|---|---|---|---|
| `postproduction.enabled` | `boolean` | `false` | Master toggle for post-processing |
| `postproduction.ao` | `boolean` | `true` | Ambient occlusion |
| `postproduction.customEdges` | `boolean` | `true` | Edge detection outlines |
| `postproduction.exclude` | `Set<THREE.Object3D>` | empty | Objects excluded from post-processing |

### Highlighter

| Property | Type | Default | Description |
|---|---|---|---|
| `multiple` | `"none" \| "shiftKey" \| "ctrlKey"` | `"none"` | Multi-selection mode |
| `zoomToSelection` | `boolean` | `false` | Auto-frame selected elements |
| `selection` | `{ [name]: ModelIdMap }` | `{}` | Current selection state |
| `styles` | `DataMap<string, MaterialDefinition>` | — | Style definitions |

Setup config:

| Config Key | Type | Default | Description |
|---|---|---|---|
| `world` | `World` | REQUIRED | World to bind to |
| `selectName` | `string` | `"select"` | Name for selection style |
| `selectionColor` | `THREE.Color` | `#BCF124` | Selection highlight color |
| `autoHighlightOnClick` | `boolean` | `true` | Auto-highlight on click |
| `selectEnabled` | `boolean` | `true` | Enable selection |

### IfcLoader

| Config Key | Type | Default | Description |
|---|---|---|---|
| `autoSetWasm` | `boolean` | `true` | Auto-resolve WASM path |
| `wasm.path` | `string` | — | Directory with WASM files |
| `wasm.absolute` | `boolean` | — | Is path absolute URL? |

### OrthoPerspectiveCamera

| Method | Parameters | Description |
|---|---|---|
| `set(mode)` | `"Orbit" \| "FirstPerson" \| "Plan"` | Switch navigation mode |
| `fit(meshes, offset?)` | meshes: `Iterable<THREE.Mesh>`, offset: `number` | Frame objects in view |

---

## TypeScript Configuration

The Vite `vanilla-ts` template provides a working `tsconfig.json`. Ensure
these compiler options are set:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

`skipLibCheck: true` is recommended because some `@thatopen` type
definitions reference internal Three.js types that may not resolve cleanly.

---

## Production Build Notes

For `vite build`:

1. WASM files from `web-ifc` MUST be accessible at runtime. Either:
   - Copy them to `public/` directory, or
   - Use CDN path via `ifcLoader.setup({ autoSetWasm: false, wasm: { ... } })`

2. COOP/COEP headers MUST be configured on the production web server.
   The Vite dev plugin does NOT affect production builds.

3. Fragment workers are bundled by Vite automatically with `worker: { format: "es" }`.
