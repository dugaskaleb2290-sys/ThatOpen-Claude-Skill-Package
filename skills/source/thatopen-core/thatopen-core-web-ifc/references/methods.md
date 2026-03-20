# IfcAPI Complete Method Signatures

## Initialization

```typescript
SetWasmPath(path: string, absolute?: boolean): void
Init(customLocateFileHandler?: LocateFileHandlerFn, forceSingleThread?: boolean): Promise<void>
Dispose(): void
```

## Model Lifecycle

```typescript
OpenModel(data: Uint8Array, settings?: LoaderSettings): number
OpenModels(dataSets: Array<Uint8Array>, settings?: LoaderSettings): Array<number>
OpenModelFromCallback(callback: ModelLoadCallback, settings?: LoaderSettings): number
CreateModel(model: NewIfcModel, settings?: LoaderSettings): number
SaveModel(modelID: number): Uint8Array
CloseModel(modelID: number): void
IsModelOpen(modelID: number): boolean
```

## Data Queries

```typescript
GetAllLines(modelID: number): Vector<number>
GetLineIDsWithType(modelID: number, type: number, includeInherited?: boolean): Vector<number>
GetLine(modelID: number, expressID: number, flatten?: boolean, inverse?: boolean, inversePropKey?: string): any
GetLines(modelID: number, expressIDs: number[], flatten?: boolean, inverse?: boolean, inversePropKey?: string): any[]
GetRawLineData(modelID: number, expressID: number): RawLineData
GetHeaderLine(modelID: number, headerType: number): any
GetLineType(modelID: number, expressID: number): number
GetMaxExpressID(modelID: number): number
GetNextExpressID(modelID: number, expressID: number): number
```

## Schema & Type Information

```typescript
GetModelSchema(modelID: number): string
GetAllTypesOfModel(modelID: number): IfcType[]
```

## Writing Data

```typescript
WriteLine<T extends IfcLineObject>(modelID: number, lineObject: T): void
WriteLines<T extends IfcLineObject>(modelID: number, lineObjects: T[]): void
DeleteLine(modelID: number, expressID: number): void
CreateIfcEntity(modelID: number, type: number, ...args: any[]): IfcLineObject
```

## Geometry Extraction

```typescript
GetFlatMesh(modelID: number, expressID: number): FlatMesh
GetGeometry(modelID: number, geometryExpressID: number): IfcGeometry
GetVertexArray(ptr: number, size: number): Float32Array
GetIndexArray(ptr: number, size: number): Uint32Array
LoadAllGeometry(modelID: number): Vector<FlatMesh>
```

## Geometry Streaming

```typescript
StreamAllMeshes(modelID: number, meshCallback: (mesh: FlatMesh, index: number, total: number) => void): void
StreamAllMeshesWithTypes(modelID: number, types: number[], meshCallback: (mesh: FlatMesh, index: number, total: number) => void): void
StreamMeshes(modelID: number, expressIDs: number[], meshCallback: (mesh: FlatMesh, index: number, total: number) => void): void
```

## Coordination & Transforms

```typescript
GetCoordinationMatrix(modelID: number): Array<number>
GetWorldTransformMatrix(modelID: number, placementExpressId: number): Array<number>
SetGeometryTransformation(modelID: number, transformationMatrix: Array<number>): void
```

## Properties Helper (ifcApi.properties)

```typescript
getItemProperties(modelID: number, id: number, recursive?: boolean, inverse?: boolean): Promise<any>
getPropertySets(modelID: number, elementID?: number, recursive?: boolean): Promise<any[]>
getTypeProperties(modelID: number, elementID?: number, recursive?: boolean): Promise<any[]>
getMaterialsProperties(modelID: number, elementID?: number, recursive?: boolean): Promise<any[]>
getSpatialStructure(modelID: number, includeProperties?: boolean): Promise<Node>
```

## GUID Utilities

```typescript
GetExpressIdFromGuid(modelID: number, guid: string): number
GetGuidFromExpressId(modelID: number, expressID: number): string
CreateIfcGuidToExpressIdMapping(modelID: number): void
```

## Logging

```typescript
SetLogLevel(level: LogLevel): void
```

---

## Type Definitions

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;    // move model geometry to origin
  USE_FAST_BOOLS?: boolean;          // faster but less accurate boolean operations
  CIRCLE_SEGMENTS_LOW?: number;      // tessellation segments for small curves
  CIRCLE_SEGMENTS_MEDIUM?: number;   // tessellation segments for medium curves
  CIRCLE_SEGMENTS_HIGH?: number;     // tessellation segments for large curves
  BOOL_ABORT_THRESHOLD?: number;     // timeout in ms for boolean operations
  MEMORY_LIMIT?: number;             // WASM memory limit in bytes
}

interface FlatMesh {
  expressID: number;
  geometries: Vector<PlacedGeometry>;
}

interface PlacedGeometry {
  color: { x: number; y: number; z: number; w: number }; // RGBA
  flatTransformation: Array<number>;   // 4x4 column-major matrix (16 floats)
  geometryExpressID: number;
}

interface IfcGeometry {
  GetVertexData(): number;       // WASM pointer to vertex data
  GetVertexDataSize(): number;   // size of vertex data
  GetIndexData(): number;        // WASM pointer to index data
  GetIndexDataSize(): number;    // size of index data
}

// WASM vector — NEVER use array indexing on this type
interface Vector<T> {
  get(index: number): T;
  size(): number;
}

interface RawLineData {
  ID: number;
  type: number;
  arguments: any[];
}

interface IfcType {
  typeID: number;
  typeName: string;
}

interface NewIfcModel {
  schema: string;       // "IFC2X3" | "IFC4" | "IFC4X3"
  name?: string;
  description?: string;
}

enum LogLevel {
  LOG_LEVEL_DEBUG = 0,
  LOG_LEVEL_INFO = 1,
  LOG_LEVEL_WARN = 2,
  LOG_LEVEL_ERROR = 3,
  LOG_LEVEL_OFF = 4,
}

// Header line type constants
const FILE_DESCRIPTION: number;
const FILE_NAME: number;
const FILE_SCHEMA: number;
```

### Vertex Data Layout

Each vertex consists of 6 floats (24 bytes):

```
[x, y, z, nx, ny, nz]
 ^-position-^  ^-normal-^
```

- `GetVertexArray` returns a `Float32Array` with `vertexCount * 6` elements
- `GetIndexArray` returns a `Uint32Array` with triangle indices (3 per triangle)
- Vertex count = `verts.length / 6`
- Triangle count = `indices.length / 3`
