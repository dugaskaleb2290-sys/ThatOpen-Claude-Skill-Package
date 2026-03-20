# web-ifc Examples

## 1. Initialize and Load an IFC File

```typescript
import * as WebIFC from "web-ifc";

async function loadIfc(url: string): Promise<{ ifcApi: WebIFC.IfcAPI; modelID: number }> {
  const ifcApi = new WebIFC.IfcAPI();
  ifcApi.SetWasmPath("/wasm/");
  await ifcApi.Init();

  const response = await fetch(url);
  const buffer = await response.arrayBuffer();
  const data = new Uint8Array(buffer);

  const modelID = ifcApi.OpenModel(data, {
    COORDINATE_TO_ORIGIN: true,
    USE_FAST_BOOLS: true,
  });

  console.log("Schema:", ifcApi.GetModelSchema(modelID));
  return { ifcApi, modelID };
}
```

## 2. Query IFC Entities by Type

```typescript
function getWalls(ifcApi: WebIFC.IfcAPI, modelID: number) {
  const wallIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);

  const walls = [];
  for (let i = 0; i < wallIDs.size(); i++) {
    const wall = ifcApi.GetLine(modelID, wallIDs.get(i));
    walls.push({
      expressID: wall.expressID,
      globalId: wall.GlobalId?.value,
      name: wall.Name?.value,
      description: wall.Description?.value,
    });
  }
  return walls;
}

// Include subtypes (e.g., IFCWALLSTANDARDCASE is a subtype of IFCWALL)
function getAllWallTypes(ifcApi: WebIFC.IfcAPI, modelID: number) {
  const wallIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL, true);
  // Returns both IFCWALL and IFCWALLSTANDARDCASE entities
  return wallIDs;
}
```

## 3. Walk the Spatial Structure

```typescript
async function printSpatialTree(ifcApi: WebIFC.IfcAPI, modelID: number) {
  const tree = await ifcApi.properties.getSpatialStructure(modelID);

  function walk(node: any, depth: number = 0) {
    const indent = "  ".repeat(depth);
    const entity = ifcApi.GetLine(modelID, node.expressID);
    console.log(`${indent}[${node.type}] ${entity.Name?.value ?? "unnamed"} (#${node.expressID})`);
    if (node.children) {
      for (const child of node.children) {
        walk(child, depth + 1);
      }
    }
  }

  walk(tree);
}
```

## 4. Extract Properties for an Element

```typescript
async function getElementProperties(ifcApi: WebIFC.IfcAPI, modelID: number, expressID: number) {
  // Basic item properties
  const item = await ifcApi.properties.getItemProperties(modelID, expressID);
  console.log("Item:", item);

  // Property sets (Psets)
  const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);
  for (const ps of psets) {
    console.log(`\nPset: ${ps.Name?.value}`);
    if (ps.HasProperties) {
      for (const prop of ps.HasProperties) {
        console.log(`  ${prop.Name?.value}: ${prop.NominalValue?.value}`);
      }
    }
  }

  // Materials
  const materials = await ifcApi.properties.getMaterialsProperties(modelID, expressID, true);
  console.log("\nMaterials:", materials);

  // Type properties
  const typeProps = await ifcApi.properties.getTypeProperties(modelID, expressID, true);
  console.log("Type:", typeProps);
}
```

## 5. Extract Geometry for a Single Element

```typescript
function extractGeometry(ifcApi: WebIFC.IfcAPI, modelID: number, expressID: number) {
  const flatMesh = ifcApi.GetFlatMesh(modelID, expressID);
  const geometries = [];

  for (let i = 0; i < flatMesh.geometries.size(); i++) {
    const pg = flatMesh.geometries.get(i);
    const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);

    const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
    const indices = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

    // Split interleaved vertex data: 6 floats per vertex [x,y,z,nx,ny,nz]
    const vertexCount = verts.length / 6;
    const positions = new Float32Array(vertexCount * 3);
    const normals = new Float32Array(vertexCount * 3);

    for (let v = 0; v < verts.length; v += 6) {
      const outIdx = (v / 6) * 3;
      positions[outIdx]     = verts[v];
      positions[outIdx + 1] = verts[v + 1];
      positions[outIdx + 2] = verts[v + 2];
      normals[outIdx]       = verts[v + 3];
      normals[outIdx + 1]   = verts[v + 4];
      normals[outIdx + 2]   = verts[v + 5];
    }

    geometries.push({
      positions,
      normals,
      indices,
      color: { r: pg.color.x, g: pg.color.y, b: pg.color.z, a: pg.color.w },
      transform: pg.flatTransformation, // 4x4 column-major
    });
  }

  return geometries;
}
```

## 6. Stream All Geometry to Three.js

```typescript
import * as WebIFC from "web-ifc";
import * as THREE from "three";

async function loadIfcToThreeJs(url: string): Promise<THREE.Group> {
  const ifcApi = new WebIFC.IfcAPI();
  ifcApi.SetWasmPath("/wasm/");
  await ifcApi.Init();

  const data = new Uint8Array(await (await fetch(url)).arrayBuffer());
  const modelID = ifcApi.OpenModel(data, { COORDINATE_TO_ORIGIN: true });

  const group = new THREE.Group();
  const coordMatrix = ifcApi.GetCoordinationMatrix(modelID);
  group.applyMatrix4(new THREE.Matrix4().fromArray(coordMatrix));

  ifcApi.StreamAllMeshes(modelID, (flatMesh, index, total) => {
    for (let i = 0; i < flatMesh.geometries.size(); i++) {
      const pg = flatMesh.geometries.get(i);
      const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);
      const v = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
      const idx = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

      // Deinterleave: 6 floats per vertex
      const pos = new Float32Array(v.length / 2);
      const norm = new Float32Array(v.length / 2);
      for (let j = 0; j < v.length; j += 6) {
        const k = (j / 6) * 3;
        pos[k] = v[j]; pos[k + 1] = v[j + 1]; pos[k + 2] = v[j + 2];
        norm[k] = v[j + 3]; norm[k + 1] = v[j + 4]; norm[k + 2] = v[j + 5];
      }

      const bufGeom = new THREE.BufferGeometry();
      bufGeom.setAttribute("position", new THREE.BufferAttribute(pos, 3));
      bufGeom.setAttribute("normal", new THREE.BufferAttribute(norm, 3));
      bufGeom.setIndex(new THREE.BufferAttribute(idx, 1));

      const { x, y, z, w } = pg.color;
      const mat = new THREE.MeshPhongMaterial({
        color: new THREE.Color(x, y, z),
        opacity: w,
        transparent: w < 1,
        side: THREE.DoubleSide,
      });

      const mesh = new THREE.Mesh(bufGeom, mat);
      mesh.applyMatrix4(new THREE.Matrix4().fromArray(pg.flatTransformation));
      group.add(mesh);
    }
  });

  ifcApi.CloseModel(modelID);
  return group;
}
```

## 7. Stream Specific Types Only

```typescript
function streamWallsAndSlabs(ifcApi: WebIFC.IfcAPI, modelID: number) {
  const targetTypes = [WebIFC.IFCWALL, WebIFC.IFCSLAB, WebIFC.IFCBEAM, WebIFC.IFCCOLUMN];

  ifcApi.StreamAllMeshesWithTypes(modelID, targetTypes, (mesh, index, total) => {
    console.log(`Processing ${index + 1}/${total}: expressID=${mesh.expressID}`);
    // Process geometry same as StreamAllMeshes callback
  });
}
```

## 8. Multi-Model Federation

```typescript
async function loadMultipleModels(urls: string[]) {
  const ifcApi = new WebIFC.IfcAPI();
  ifcApi.SetWasmPath("/wasm/");
  await ifcApi.Init();

  const models: Array<{ modelID: number; matrix: number[] }> = [];

  for (const url of urls) {
    const data = new Uint8Array(await (await fetch(url)).arrayBuffer());
    const modelID = ifcApi.OpenModel(data, { COORDINATE_TO_ORIGIN: true });
    const matrix = ifcApi.GetCoordinationMatrix(modelID);
    models.push({ modelID, matrix });
    console.log(`Loaded model ${modelID}: schema=${ifcApi.GetModelSchema(modelID)}`);
  }

  // Process all models...

  // ALWAYS close all models when done
  for (const { modelID } of models) {
    ifcApi.CloseModel(modelID);
  }
}
```

## 9. GUID Lookups

```typescript
function guidExamples(ifcApi: WebIFC.IfcAPI, modelID: number) {
  // Pre-build mapping for performance (do this once)
  ifcApi.CreateIfcGuidToExpressIdMapping(modelID);

  // GUID to expressID
  const expressID = ifcApi.GetExpressIdFromGuid(modelID, "3Dr$t5gOX0AxMQNRw6q2K$");
  console.log("ExpressID:", expressID);

  // expressID to GUID
  const guid = ifcApi.GetGuidFromExpressId(modelID, expressID);
  console.log("GUID:", guid);
}
```

## 10. Create and Save a New IFC Model

```typescript
async function createNewModel(ifcApi: WebIFC.IfcAPI) {
  const modelID = ifcApi.CreateModel({
    schema: "IFC4",
    name: "New Model",
    description: "Created with web-ifc",
  });

  // Create entities using CreateIfcEntity...
  // Write data using WriteLine...

  const ifcData = ifcApi.SaveModel(modelID);
  const blob = new Blob([ifcData], { type: "application/octet-stream" });

  ifcApi.CloseModel(modelID);
  return blob;
}
```

## 11. Discover Model Contents

```typescript
function discoverModel(ifcApi: WebIFC.IfcAPI, modelID: number) {
  // What schema?
  console.log("Schema:", ifcApi.GetModelSchema(modelID));

  // What types are present?
  const types = ifcApi.GetAllTypesOfModel(modelID);
  for (const t of types) {
    const count = ifcApi.GetLineIDsWithType(modelID, t.typeID).size();
    console.log(`  ${t.typeName}: ${count} entities`);
  }

  // Total entities
  const allLines = ifcApi.GetAllLines(modelID);
  console.log(`Total entities: ${allLines.size()}`);

  // Header info
  const fileName = ifcApi.GetHeaderLine(modelID, WebIFC.FILE_NAME);
  console.log("File name:", fileName);
}
```
