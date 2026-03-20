# API Signatures — Core Architecture

Version: @thatopen/components 3.3.x

---

## Components Container

```typescript
class Components implements Disposable {
  readonly list: DataMap<string, Component>;
  enabled: boolean;
  onDisposed: Event<void>;
  onInit: Event<undefined>;
  static release: string; // version string

  /**
   * Lazy singleton getter. Creates the component on first call,
   * returns the cached instance on subsequent calls.
   */
  get<U extends Component>(Ctor: new (c: Components) => U): U;

  /**
   * Registers a component instance by UUID.
   * Called internally by component constructors.
   */
  add(uuid: string, instance: Component): void;

  /**
   * Starts the requestAnimationFrame loop.
   * Uses THREE.Clock for delta time calculation.
   * Calls update(delta) on all enabled Updateable components each frame.
   */
  init(): void;

  /**
   * Disposes all registered components.
   * FragmentsManager is ALWAYS disposed last.
   */
  dispose(): void;
}
```

## Base Class

```typescript
abstract class Base {
  constructor(public components: Components) {}

  /** Runtime check: does this implement Disposable? */
  isDisposeable(): this is Disposable;

  /** Runtime check: does this implement Updateable? */
  isUpdateable(): this is Updateable;

  /** Runtime check: does this implement Configurable? */
  isConfigurable(): this is Configurable<any, any>;

  /** Runtime check: does this implement Resizeable? */
  isResizeable(): this is Resizeable;

  /** Runtime check: does this implement Hideable? */
  isHideable(): this is Hideable;

  /** Runtime check: does this implement Serializable? */
  isSerializable(): this is Serializable<any>;
}
```

## Component Class

```typescript
abstract class Component extends Base {
  /** MUST be unique across all components. Used as registry key. */
  static readonly uuid: string;

  /** Whether this component is active. */
  abstract enabled: boolean;
}
```

## Lifecycle Interfaces

### Disposable

```typescript
interface Disposable {
  dispose(): void;
  onDisposed: Event<void>;
}
```

### Updateable

```typescript
interface Updateable {
  update(delta?: number): void;
  onBeforeUpdate: Event<any>;
  onAfterUpdate: Event<any>;
}
```

### Configurable

```typescript
interface Configurable<TConfig, TPartialConfig> {
  setup(config?: TPartialConfig): void;
  config: TConfig;
  isSetup: boolean;
  onSetup: Event<any>;
}
```

### Resizeable

```typescript
interface Resizeable {
  resize(size?: THREE.Vector2): void;
  getSize(): THREE.Vector2;
  onResize: Event<THREE.Vector2>;
}
```

### Hideable

```typescript
interface Hideable {
  visible: boolean;
}
```

### Createable

```typescript
interface Createable {
  create(data?: any): void;
  delete(data?: any): void;
  endCreation(data?: any): void;
  cancelCreation(data?: any): void;
}
```

### Serializable

```typescript
interface Serializable<T> {
  import(data: T): void;
  export(): T;
}
```

### CameraControllable

```typescript
interface CameraControllable {
  controls: CameraControls;
}
```

### Eventable

```typescript
interface Eventable {
  eventManager: EventManager;
}
```

## Worlds Manager

```typescript
class Worlds extends Component implements Updateable, Disposable {
  static readonly uuid: string;
  enabled: boolean;
  list: DataMap<string, World>;

  /** Creates a new World with typed scene, camera, renderer. */
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
  meshes: Set<THREE.Mesh>;

  onDisposed: Event<void>;

  dispose(): void;
}
```

## SimpleScene

```typescript
class SimpleScene extends BaseScene implements Configurable {
  three: THREE.Scene;

  /** Creates default directional + ambient lights and sets background. */
  setup(config?: Partial<SimpleSceneConfig>): void;

  config: SimpleSceneConfig;
  isSetup: boolean;
  onSetup: Event<SimpleScene>;
}
```

## SimpleRenderer

```typescript
class SimpleRenderer extends BaseRenderer implements Disposable, Resizeable, Updateable {
  three: THREE.WebGLRenderer;

  constructor(components: Components, container: HTMLElement, parameters?: Partial<THREE.WebGLRendererParameters>);

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

## OrthoPerspectiveCamera

```typescript
class OrthoPerspectiveCamera extends BaseCamera
  implements CameraControllable, Updateable, Disposable, Configurable {

  three: THREE.PerspectiveCamera | THREE.OrthographicCamera;
  threePersp: THREE.PerspectiveCamera;
  threeOrtho: THREE.OrthographicCamera;
  controls: CameraControls;
  projection: ProjectionManager;

  /** Switch navigation mode: "Orbit" | "FirstPerson" | "Plan" */
  set(mode: NavigationMode): void;

  /** Frame objects in view */
  fit(meshes?: Iterable<THREE.Mesh>, offset?: number): void;

  /** Add a custom navigation mode */
  addCustomNavigationMode(mode: NavigationMode): void;

  setup(config?: any): void;
  update(delta?: number): void;
  dispose(): void;

  onDisposed: Event<void>;
}
```

## Event

```typescript
class Event<T> {
  enabled: boolean;

  /** Subscribe a handler. */
  add(handler: (data: T) => void): void;

  /** Unsubscribe a handler. */
  remove(handler: (data: T) => void): void;

  /** Fire the event, calling all handlers with the provided data. */
  trigger(data?: T): void;

  /** Remove all handlers. */
  reset(): void;
}
```

## DataMap

```typescript
class DataMap<K, V> extends Map<K, V> {
  onItemSet: Event<{ key: K; value: V }>;
  onItemUpdated: Event<{ key: K; value: V }>;
  onItemDeleted: Event<{ key: K }>;
  onCleared: Event<void>;
}
```

## DataSet

```typescript
class DataSet<T> extends Set<T> {
  onItemAdded: Event<T>;
  onItemDeleted: Event<T>;
  onCleared: Event<void>;
}
```

## ModelIdMap

```typescript
/**
 * Universal item targeting structure.
 * Keys: model IDs (strings).
 * Values: sets of local element IDs (numbers).
 */
type ModelIdMap = Record<string, Set<number>>;
```
