# API Signatures — Component Syntax

Version: @thatopen/components 3.3.x

---

## Components Container

```typescript
class Components implements Disposable {
  readonly list: DataMap<string, Component>;
  enabled: boolean;
  onDisposed: Event<void>;
  onInit: Event<undefined>;
  static release: string;

  /**
   * Lazy singleton getter. Creates the component on first call,
   * returns the cached instance on subsequent calls.
   * The component's constructor is invoked with this Components instance.
   */
  get<U extends Component>(Ctor: new (c: Components) => U): U;

  /**
   * Registers a component instance by UUID.
   * Called internally by component constructors — NEVER call manually
   * outside of a component constructor.
   */
  add(uuid: string, instance: Component): void;

  /**
   * Starts the requestAnimationFrame loop.
   * Uses THREE.Clock for delta time calculation.
   * Calls update(delta) on all enabled Updateable components each frame.
   * Triggers onInit event.
   */
  init(): void;

  /**
   * Disposes all registered components.
   * FragmentsManager is ALWAYS disposed last.
   * Triggers onDisposed event.
   * After this call, all references are invalidated.
   */
  dispose(): void;
}
```

## Base Class

```typescript
abstract class Base {
  constructor(public components: Components) {}

  /** Runtime check: does this implement Disposable? (duck-typing) */
  isDisposeable(): this is Disposable;

  /** Runtime check: does this implement Updateable? (duck-typing) */
  isUpdateable(): this is Updateable;

  /** Runtime check: does this implement Configurable? (duck-typing) */
  isConfigurable(): this is Configurable<any, any>;

  /** Runtime check: does this implement Resizeable? (duck-typing) */
  isResizeable(): this is Resizeable;

  /** Runtime check: does this implement Hideable? (duck-typing) */
  isHideable(): this is Hideable;

  /** Runtime check: does this implement Serializable? (duck-typing) */
  isSerializable(): this is Serializable<any>;
}
```

## Component Class

```typescript
abstract class Component extends Base {
  /** MUST be unique across all components. Used as registry key. */
  static readonly uuid: string;

  /** Whether this component is active. MUST be implemented. */
  abstract enabled: boolean;
}
```

## Event\<T\>

```typescript
class Event<T> {
  /**
   * Whether the event is active. When false, trigger() is a no-op.
   * Handlers remain registered but do not fire.
   * Default: true
   */
  enabled: boolean;

  /**
   * Subscribe a handler function.
   * The handler will be called with the event data when triggered.
   * Duplicate handlers are allowed — the same function can be added
   * multiple times and will fire multiple times.
   */
  add(handler: (data: T) => void): void;

  /**
   * Unsubscribe a handler function.
   * Requires the SAME function reference that was passed to add().
   * If the handler was added multiple times, only the first match
   * is removed.
   */
  remove(handler: (data: T) => void): void;

  /**
   * Fire the event, calling all registered handlers with the provided data.
   * If enabled is false, this is a no-op.
   * Handlers are called synchronously in registration order.
   */
  trigger(data?: T): void;

  /**
   * Remove ALL registered handlers.
   * ALWAYS call this in dispose() to prevent memory leaks.
   */
  reset(): void;
}
```

## Disposable Interface

```typescript
interface Disposable {
  /** Clean up all resources. Set enabled=false, trigger onDisposed. */
  dispose(): void;

  /** Fired when dispose() is called. Reset this in dispose(). */
  onDisposed: Event<void>;
}
```

## Updateable Interface

```typescript
interface Updateable {
  /**
   * Called by the Components animation loop every frame.
   * Only called when enabled=true.
   * @param delta — seconds since the last frame (from THREE.Clock)
   */
  update(delta?: number): void;

  /** Fired at the start of update(), before logic runs. */
  onBeforeUpdate: Event<any>;

  /** Fired at the end of update(), after logic completes. */
  onAfterUpdate: Event<any>;
}
```

## Configurable Interface

```typescript
interface Configurable<TConfig, TPartialConfig> {
  /**
   * Initialize or reconfigure the component.
   * ALWAYS call before using the component's main functionality.
   * @param config — partial config merged with defaults
   */
  setup(config?: TPartialConfig): void;

  /** Current configuration. Read-only after setup. */
  config: TConfig;

  /** Whether setup() has been called. Check before using the component. */
  isSetup: boolean;

  /** Fired when setup() completes successfully. */
  onSetup: Event<any>;
}
```

## Resizeable Interface

```typescript
interface Resizeable {
  /** Resize the component (e.g., renderer viewport). */
  resize(size?: THREE.Vector2): void;

  /** Get the current size. */
  getSize(): THREE.Vector2;

  /** Fired when resize occurs. */
  onResize: Event<THREE.Vector2>;
}
```

## Hideable Interface

```typescript
interface Hideable {
  /** Whether the component is visible. */
  visible: boolean;
}
```

## Createable Interface

```typescript
interface Createable {
  /** Create a new item (e.g., measurement, clipping plane). */
  create(data?: any): void;

  /** Delete an item. */
  delete(data?: any): void;

  /** Finish an interactive creation session. */
  endCreation(data?: any): void;

  /** Cancel an interactive creation session. */
  cancelCreation(data?: any): void;
}
```

## Serializable Interface

```typescript
interface Serializable<T> {
  /** Import data into the component. */
  import(data: T): void;

  /** Export the component's state as data. */
  export(): T;
}
```

## DataMap\<K, V\>

```typescript
class DataMap<K, V> extends Map<K, V> {
  /**
   * Fired when a NEW key-value pair is added via set().
   * Payload: the key and value that were added.
   */
  onItemSet: Event<{ key: K; value: V }>;

  /**
   * Fired when an EXISTING key's value is updated via set().
   * Payload: the key and new value.
   */
  onItemUpdated: Event<{ key: K; value: V }>;

  /**
   * Fired when a key-value pair is removed via delete().
   * Payload: the key that was removed.
   */
  onItemDeleted: Event<{ key: K }>;

  /**
   * Fired when clear() is called, removing all entries.
   */
  onCleared: Event<void>;
}

// Inherits all Map methods:
// get(key), set(key, value), delete(key), has(key), clear()
// forEach(), entries(), keys(), values(), size
```

## DataSet\<T\>

```typescript
class DataSet<T> extends Set<T> {
  /**
   * Fired when a new item is added via add().
   * Payload: the item that was added.
   */
  onItemAdded: Event<T>;

  /**
   * Fired when an item is removed via delete().
   * Payload: the item that was removed.
   */
  onItemDeleted: Event<T>;

  /**
   * Fired when clear() is called, removing all items.
   */
  onCleared: Event<void>;
}

// Inherits all Set methods:
// add(value), delete(value), has(value), clear()
// forEach(), entries(), keys(), values(), size
```
