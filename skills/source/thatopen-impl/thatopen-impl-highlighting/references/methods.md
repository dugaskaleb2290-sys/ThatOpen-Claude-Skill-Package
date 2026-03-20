# Highlighting API Reference

## Types

### MaterialDefinition

```typescript
type MaterialDefinition = {
  color: THREE.Color;
  opacity: number;
};
```

### ModelIdMap

```typescript
type ModelIdMap = Map<string, Set<number>>;
// Maps model UUID → set of local element IDs
```

### HighlighterConfig

```typescript
interface HighlighterConfig {
  world: World;
  selectName: string;                         // default: "select"
  selectionColor: THREE.Color;                // default: #BCF124
  autoHighlightOnClick: boolean;              // default: true
  selectEnabled: boolean;                     // default: true
  autoUpdateFragments: boolean;               // default: true
  selectMaterialDefinition: MaterialDefinition; // default: yellow
}
```

---

## Highlighter

**Package**: `@thatopen/components-front`
**Implements**: `Component`, `Disposable`, `Eventable`

### Properties

| Property | Type | Description |
|---|---|---|
| `multiple` | `"none" \| "shiftKey" \| "ctrlKey"` | Multi-selection modifier key |
| `zoomToSelection` | `boolean` | Auto-zoom camera to selection |
| `selection` | `{ [styleName: string]: ModelIdMap }` | Current selections per style |
| `styles` | `DataMap<string, MaterialDefinition \| null>` | Registered highlight styles |
| `autoToggle` | `Set<string>` | Style names that toggle on re-click |
| `selectable` | `{ [name: string]: ModelIdMap }` | Per-style selectable element filter |

### Methods

#### setup(config?)

Initializes the Highlighter for a specific world. MUST be called before
any other method.

```typescript
setup(config?: Partial<HighlighterConfig>): void
```

- Creates the default `"select"` style from `selectionColor` /
  `selectMaterialDefinition`.
- Binds click event listener to the world's renderer container.
- Sets `autoHighlightOnClick` behavior.

#### highlight(name, removePrevious?, zoomToSelection?, exclude?)

Highlights the element currently under the cursor using the specified style.

```typescript
highlight(
  name: string,
  removePrevious?: boolean,   // default: true
  zoomToSelection?: boolean,  // default: false
  exclude?: ModelIdMap
): Promise<void>
```

- Uses raycasting from the world's camera to determine which element is
  under the cursor.
- When `removePrevious` is `true`, clears the style's current selection
  before highlighting.
- When `zoomToSelection` is `true`, frames the camera on the selected
  elements.
- `exclude` prevents specific elements from being highlighted.

#### highlightByID(name, modelIdMap, removePrevious?, zoomToSelection?, exclude?)

Highlights specific elements by their IDs without requiring a mouse event.

```typescript
highlightByID(
  name: string,
  modelIdMap: ModelIdMap,
  removePrevious?: boolean,   // default: true
  zoomToSelection?: boolean,  // default: false
  exclude?: ModelIdMap
): Promise<void>
```

#### clear(name?, filter?)

Removes highlighting from elements.

```typescript
clear(name?: string, filter?: ModelIdMap): Promise<void>
```

- Without arguments: clears ALL styles.
- With `name`: clears only the specified style.
- With `filter`: removes only the specified elements from the style.

#### updateColors()

Refreshes the visual appearance of all highlighted elements. Call after
modifying a style's MaterialDefinition.

```typescript
updateColors(): Promise<void>
```

---

## Hoverer

**Package**: `@thatopen/components-front`

### Properties

| Property | Type | Description |
|---|---|---|
| `duration` | `number` | Fade animation duration in milliseconds |
| `delay` | `number` | Delay before hover effect triggers (ms) |
| `animation` | `string` | Animation type identifier |

### Events

| Event | Payload | Description |
|---|---|---|
| `onHoverStarted` | hover data | Fires when cursor enters an element |
| `onHoverEnded` | void | Fires when cursor leaves an element |

---

## Outliner

**Package**: `@thatopen/components-front`
**Requires**: `PostproductionRenderer` with `postproduction.enabled = true`

### Properties

| Property | Type | Description |
|---|---|---|
| `styles` | `DataSet<string>` | Highlighter style names to outline |
| `color` | `THREE.Color` | Outline stroke color |
| `thickness` | `number` | Outline stroke width in pixels |
| `fillColor` | `THREE.Color` | Interior fill color |
| `fillOpacity` | `number` | Interior fill opacity (0-1) |

### Methods

#### addItems(modelIdMap)

Manually add elements to outline rendering.

```typescript
addItems(modelIdMap: ModelIdMap): void
```

#### removeItems(modelIdMap)

Remove specific elements from outline rendering.

```typescript
removeItems(modelIdMap: ModelIdMap): void
```

#### clean()

Remove all items from outline rendering.

```typescript
clean(): void
```

---

## Mesher

**Package**: `@thatopen/components-front`

### Methods

#### get(modelIdMap, config?)

Converts fragment elements into standard THREE.Mesh objects.

```typescript
get(modelIdMap: ModelIdMap, config?: MesherConfig): THREE.Mesh[]
```

- Returns an array of `THREE.Mesh` objects with geometry and material.
- Returned meshes are NOT managed by the fragment system.
- Caller is responsible for adding meshes to a scene and disposing them.

---

## FastModelPicker

**Package**: `@thatopen/components`

GPU-based element picking alternative to raycasting. Renders element IDs
to an off-screen buffer and reads the pixel under the cursor.

- New in v3.
- Significantly faster than raycasting for models with 100k+ elements.
- For standard use cases, the Highlighter's built-in raycasting is
  sufficient and simpler.
