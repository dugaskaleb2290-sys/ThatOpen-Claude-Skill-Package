# API Methods — Viewer Setup

Version: @thatopen/components 3.3.x, @thatopen/components-front 3.3.x

---

## Worlds Manager

```typescript
class Worlds extends Component implements Updateable, Disposable {
  static readonly uuid: string;
  enabled: boolean;
  list: DataMap<string, World>;

  /**
   * Creates a new World. Type parameters are for TypeScript narrowing only.
   * You MUST still assign scene, camera, renderer manually.
   */
  create<TScene, TCamera, TRenderer>(): SimpleWorld;

  /** Disposes and removes a world from the list. */
  delete(world: World): void;

  /** Called by Components animation loop. Updates all worlds. */
  update(delta?: number): void;

  dispose(): void;
  onDisposed: Event<void>;
}
```

## SimpleWorld

```typescript
class SimpleWorld implements World, Disposable {
  uuid: string;
  scene: BaseScene;
  camera: BaseCamera;
  renderer: BaseRenderer;

  /** Set of meshes tracked for raycasting. */
  meshes: Set<THREE.Mesh>;

  onDisposed: Event<void>;
  dispose(): void;
}
```

## SimpleScene

```typescript
class SimpleScene extends BaseScene implements Configurable {
  /** The underlying Three.js scene. */
  three: THREE.Scene;

  /**
   * Creates default lighting and background.
   * Adds a DirectionalLight and an AmbientLight to the scene.
   * Sets a default background color.
   */
  setup(config?: Partial<SimpleSceneConfig>): void;

  config: SimpleSceneConfig;
  isSetup: boolean;
  onSetup: Event<SimpleScene>;

  dispose(): void;
  onDisposed: Event<void>;
}
```

### SimpleSceneConfig

```typescript
interface SimpleSceneConfig {
  directionalLight: {
    color: THREE.Color;
    intensity: number;
    position: THREE.Vector3;
  };
  ambientLight: {
    color: THREE.Color;
    intensity: number;
  };
}
```

## SimpleRenderer

```typescript
class SimpleRenderer extends BaseRenderer
  implements Disposable, Resizeable, Updateable {

  /** The underlying Three.js WebGLRenderer. */
  three: THREE.WebGLRenderer;

  /**
   * @param components - Components container
   * @param container - DOM element to render into
   * @param parameters - Optional THREE.WebGLRendererParameters
   */
  constructor(
    components: Components,
    container: HTMLElement,
    parameters?: Partial<THREE.WebGLRendererParameters>
  );

  /** Resize to given size or auto-detect from container. */
  resize(size?: THREE.Vector2): void;

  /** Get current renderer size. */
  getSize(): THREE.Vector2;

  /** Render one frame. Called by animation loop. */
  update(): void;

  dispose(): void;

  onResize: Event<THREE.Vector2>;
  onDisposed: Event<void>;
  onBeforeUpdate: Event<any>;
  onAfterUpdate: Event<any>;
}
```

### WebGLRendererParameters (subset)

```typescript
interface WebGLRendererParameters {
  antialias?: boolean;    // Default: false. Enable MSAA.
  alpha?: boolean;        // Default: false. Transparent background.
  powerPreference?: "high-performance" | "low-power" | "default";
  preserveDrawingBuffer?: boolean; // Needed for screenshots.
}
```

## PostproductionRenderer

Package: `@thatopen/components-front` (browser-only)

```typescript
class PostproductionRenderer extends RendererWith2D
  implements Disposable, Resizeable, Updateable {

  /** The underlying Three.js WebGLRenderer. */
  three: THREE.WebGLRenderer;

  /**
   * @param components - Components container
   * @param container - DOM element to render into
   * @param parameters - Optional WebGL parameters
   */
  constructor(
    components: Components,
    container: HTMLElement,
    parameters?: Partial<THREE.WebGLRendererParameters>
  );

  /** Post-processing controller. */
  postproduction: Postproduction;

  resize(size?: THREE.Vector2): void;
  getSize(): THREE.Vector2;
  update(): void;
  dispose(): void;

  onResize: Event<THREE.Vector2>;
  onDisposed: Event<void>;
  onBeforeUpdate: Event<any>;
  onAfterUpdate: Event<any>;
}
```

### Postproduction

```typescript
class Postproduction {
  /** Enable/disable all post-processing. Default: false. */
  enabled: boolean;

  /** Enable/disable ambient occlusion. */
  ao: boolean;

  /** Enable/disable custom edge outlines. */
  customEdges: boolean;

  /** Enable/disable gloss pass. */
  glossEnabled: boolean;

  /** Set of Three.js objects to exclude from post-processing. */
  exclude: Set<THREE.Object3D>;
}
```

## OrthoPerspectiveCamera

```typescript
class OrthoPerspectiveCamera extends BaseCamera
  implements CameraControllable, Updateable, Disposable, Configurable {

  /** Currently active Three.js camera (perspective or orthographic). */
  three: THREE.PerspectiveCamera | THREE.OrthographicCamera;

  /** The perspective camera instance. */
  threePersp: THREE.PerspectiveCamera;

  /** The orthographic camera instance. */
  threeOrtho: THREE.OrthographicCamera;

  /** camera-controls instance for orbit/pan/zoom. */
  controls: CameraControls;

  /** Projection manager for switching perspective/ortho. */
  projection: ProjectionManager;

  /**
   * Switch navigation mode.
   * @param mode - "Orbit" | "FirstPerson" | "Plan"
   */
  set(mode: NavigationMode): void;

  /**
   * Frame objects in view.
   * @param meshes - Meshes to frame (defaults to all world meshes)
   * @param offset - Zoom offset factor (1.0 = tight fit)
   */
  fit(meshes?: Iterable<THREE.Mesh>, offset?: number): void;

  /** Register a custom navigation mode. */
  addCustomNavigationMode(mode: NavigationMode): void;

  setup(config?: any): void;
  update(delta?: number): void;
  dispose(): void;

  onDisposed: Event<void>;
}
```

### NavigationMode

```typescript
type NavigationMode = "Orbit" | "FirstPerson" | "Plan" | string;
```

- **Orbit**: Default. Rotate around a target point with mouse/touch.
- **FirstPerson**: WASD-style movement with mouse look.
- **Plan**: Orthographic top-down view. Switches to ortho projection.

## Grids

```typescript
class Grids extends Component implements Disposable {
  static readonly uuid: string;
  enabled: boolean;

  /** Create a grid for the given world. */
  create(world: World): SimpleGrid;

  dispose(): void;
  onDisposed: Event<void>;
}
```

### SimpleGrid

```typescript
class SimpleGrid implements Disposable, Hideable {
  /** The underlying Three.js grid mesh. */
  three: THREE.Mesh;

  visible: boolean;

  dispose(): void;
  onDisposed: Event<void>;
}
```

## Components Lifecycle Methods (Relevant)

```typescript
class Components {
  /**
   * Start the requestAnimationFrame loop.
   * Calls update(delta) on all enabled Updateable components each frame.
   */
  init(): void;

  /**
   * Stop the loop and dispose ALL registered components.
   * FragmentsManager is disposed last.
   */
  dispose(): void;

  onInit: Event<undefined>;
  onDisposed: Event<void>;
}
```
