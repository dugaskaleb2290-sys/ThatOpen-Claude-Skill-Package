# IfcLoader & IfcImporter — API Reference

## IfcLoader (from @thatopen/components)

### Class Definition

```typescript
class IfcLoader extends Component implements Disposable, Configurable<IfcFragmentSettings> {
  static readonly uuid: string;  // "a659add7-1418-4771-a0d6-7d4d438e4624"
  enabled: boolean;              // default: true
  settings: IfcFragmentSettings;
  webIfc: WEBIFC.IfcAPI;

  // Events
  onDisposed: Event<string>;
  onSetup: Event<void>;
  onIfcStartedLoading: Event<void>;
  onIfcImporterInitialized: Event<IfcImporter>;
}
```

### Methods

#### `setup(config?: Partial<IfcFragmentSettings>): Promise<void>`

Initializes the IfcLoader with WASM configuration. MUST be called before `load()`.

- When `autoSetWasm` is `true` (default), automatically resolves the WASM path from the installed `web-ifc` package.
- When `autoSetWasm` is `false`, uses `wasm.path` and `wasm.absolute` from the provided config.
- Fires `onSetup` event on completion.

```typescript
// Auto WASM (default)
await ifcLoader.setup();

// Manual WASM
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "https://unpkg.com/web-ifc@0.0.77/", absolute: true },
});
```

#### `load(data: Uint8Array, coordinate: boolean, name: string, config?: object): Promise<FragmentsModel>`

Converts an IFC file into a FragmentsModel for 3D rendering.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `data` | `Uint8Array` | Yes | IFC file contents |
| `coordinate` | `boolean` | Yes | Apply coordination matrix for multi-model alignment |
| `name` | `string` | Yes | Display name for the model |
| `config` | `object` | No | `{ instanceCallback?, processData?, userData? }` |

Returns: `Promise<FragmentsModel>` — The loaded model, ready to add to a scene.

Fires `onIfcStartedLoading` at the beginning of parsing.
Fires `onIfcImporterInitialized` when the internal IfcImporter is created (use this to customize import classes).

#### `readIfcFile(data: Uint8Array): Promise<number>`

Opens an IFC file in the internal web-ifc instance without converting to fragments. Returns the `modelID` for direct web-ifc API access via `ifcLoader.webIfc`.

ALWAYS call `cleanUp()` after using `readIfcFile()`.

#### `cleanUp(): void`

Resets the internal web-ifc state and clears maps. Called automatically after `load()`. Call manually after `readIfcFile()`.

#### `dispose(): void`

Releases all resources including the web-ifc WASM instance.

---

## IfcFragmentSettings

```typescript
interface IfcFragmentSettings {
  autoSetWasm: boolean;              // default: true
  wasm: {
    path: string;                    // directory containing WASM files
    absolute: boolean;               // true = absolute URL, false = relative
    logLevel?: LogLevel;             // web-ifc log verbosity
  };
  webIfc: LoaderSettings;            // passed to web-ifc OpenModel()
  customLocateFileHandler: LocateFileHandlerFn | null;  // default: null
}
```

### LoaderSettings (web-ifc)

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;    // translate geometry to origin
  USE_FAST_BOOLS?: boolean;          // faster but less accurate booleans
  CIRCLE_SEGMENTS_LOW?: number;      // tessellation: small curves
  CIRCLE_SEGMENTS_MEDIUM?: number;   // tessellation: medium curves
  CIRCLE_SEGMENTS_HIGH?: number;     // tessellation: large curves
  BOOL_ABORT_THRESHOLD?: number;     // timeout (ms) for boolean ops
  MEMORY_LIMIT?: number;             // WASM memory limit in bytes
}
```

---

## IfcImporter (from @thatopen/fragments)

### Class Definition

```typescript
class IfcImporter {
  wasm: { path: string; absolute: boolean };
  classes: {
    elements: Set<number>;           // physical IFC element types to import
    abstract: Set<number>;           // abstract types (materials, properties)
  };
  relations: Map<number, { forRelating: number[]; forRelated: number[] }>;
  distanceThreshold: number | null;  // default: 100000
  includeRelationNames: boolean;     // default: false
  includeUniqueAttributes: boolean;  // default: false
  replaceSiteElevation: boolean;     // default: true
  replaceStoreyElevation: boolean;   // default: true
  attributesToExclude: Set<string>;
}
```

### Methods

#### `process(data: Uint8Array): Promise<Uint8Array>`

Converts an IFC file to Fragments binary format. Input is IFC file bytes, output is Fragments binary bytes.

#### `addAllAttributes(): void`

Populates the importer to include all IFC attributes in the output. Use cautiously — significantly increases output size.

#### `addAllRelations(): void`

Populates the importer to include all IFC relations in the output. Use cautiously — significantly increases output size.

---

## FragmentsModel (return type of load())

Key properties and methods of the loaded model:

```typescript
class FragmentsModel {
  object: THREE.Object3D;            // add this to your scene
  export(): Uint8Array;              // serialize to Fragments binary
  dispose(): void;                   // free GPU and CPU resources
}
```

---

## Event Types

| Event | Payload | Description |
|---|---|---|
| `onSetup` | `void` | Fired after `setup()` completes |
| `onIfcStartedLoading` | `void` | Fired when IFC parsing begins in `load()` |
| `onIfcImporterInitialized` | `IfcImporter` | Fired when the IfcImporter instance is created; use to modify `classes` |
| `onDisposed` | `string` | Fired when the component is disposed |

### Event Usage Pattern

```typescript
// Listen for setup completion
ifcLoader.onSetup.add(() => {
  console.log("IfcLoader ready");
});

// Customize import classes before conversion begins
ifcLoader.onIfcImporterInitialized.add((importer) => {
  importer.classes.elements.add(WEBIFC.IFCSPACE);
  importer.classes.elements.delete(WEBIFC.IFCFURNISHINGELEMENT);
});

// Track loading progress
ifcLoader.onIfcStartedLoading.add(() => {
  console.log("IFC parsing started...");
});
```
