# API Signatures — UI Component Syntax

Version: @thatopen/ui 3.3.x / @thatopen/ui-obc 3.3.x

---

## BUI.Manager

```typescript
class Manager {
  /**
   * Registers all bim-* custom elements and injects global styles.
   * MUST be called once before any bim-* element is used.
   * @param querySelectorElements — CSS selector for animation targets (optional)
   * @param animateOnLoad — enable entrance animations (default: true)
   */
  static init(
    querySelectorElements?: string,
    animateOnLoad?: boolean
  ): void;

  /**
   * Toggles between bim-ui-dark and bim-ui-light CSS classes on <html>.
   * @param animate — show overlay transition animation (default: false)
   */
  static toggleTheme(animate?: boolean): void;

  /**
   * Preloads Iconify icon collections for offline use.
   * @param collections — array of collection names (e.g., ["mdi", "material-symbols"])
   * @param log — log loading status to console (default: false)
   */
  static preloadIcons(collections: string[], log?: boolean): Promise<void>;

  /** Injects global BIM UI stylesheet into document head (idempotent). */
  static addGlobalStyles(): void;

  /** Generates a random 10-character alphanumeric ID. */
  static newRandomId(): string;

  /** Runtime configuration. */
  static config: ManagerConfig;
}

interface ManagerConfig {
  /** Show section labels on vertical toolbars. Default: false. */
  sectionLabelOnVerticalToolbar: boolean;
  /** Attribute name for internal component identification. */
  internalComponentNameAttribute: string;
}
```

---

## Layout Components

### bim-grid

```typescript
// Custom element: <bim-grid>
interface BimGrid extends LitElement {
  /** Name of the active layout from the layouts collection. */
  layout: string;                        // reflects

  /** Whether the grid floats (absolute position, no pointer-events on gaps). */
  floating: boolean;                     // reflects, default: false

  /** Enable interactive resize handles between grid areas. */
  resizeableAreas: boolean;              // attribute: areas-resizeable, default: false

  /** Areas excluded from resizing when resizeableAreas is true. */
  areasResizeExceptions: string[];

  /** Collection of named layout definitions. */
  layouts: GridLayoutsDefinition;

  /** Map of area names to HTML elements. */
  elements: GridComponents;

  /** State update functions for stateful elements. */
  updateComponent: UpdateGridComponents;

  /** Stored resize dimensions per layout (after user resizing). */
  layoutsResize: Record<string, object>;
}

// Events
"layoutchange"    // layout property changed
"elementcreated"  // element instantiated for an area

interface GridLayoutsDefinition {
  [name: string]: {
    template: string;           // CSS grid-template value
    elements: GridComponents;   // area name -> HTMLElement
    guard?: () => boolean;      // conditional rendering
  };
}
```

### bim-panel

```typescript
// Custom element: <bim-panel>
interface BimPanel extends LitElement {
  icon: string;                          // reflects
  name: string;                          // reflects
  label: string;                         // reflects
  hidden: boolean;                       // reflects, default: false
  headerHidden: boolean;                 // reflects, default: false
  value: Record<string, any>;            // aggregated form values (readonly)
  valueTransform: Record<string, (v: any) => any>;
  activationButton: BimButton;           // auto-created toggle button
}

// Events
"change"        // form value changed in any child element
"hiddenchange"  // hidden property toggled

// Slots
// default — accepts bim-panel-section elements
```

### bim-panel-section

```typescript
// Custom element: <bim-panel-section>
interface BimPanelSection extends LitElement {
  label: string;                         // reflects
  icon: string;                          // reflects
  collapsed: boolean;                    // reflects, default: false
  fixed: boolean;                        // reflects — prevents collapsing
}

// Slots
// default — accepts any form elements or content
```

### bim-toolbar

```typescript
// Custom element: <bim-toolbar>
interface BimToolbar extends LitElement {
  icon: string;                          // reflects
  vertical: boolean;                     // reflects, default: false
  labelsHidden: boolean;                 // reflects, default: false
  hidden: boolean;                       // reflects, default: false
}

// Events
"hiddenchange"  // hidden property toggled

// Slots
// default — accepts bim-toolbar-section elements
```

### bim-toolbar-section

```typescript
// Custom element: <bim-toolbar-section>
interface BimToolbarSection extends LitElement {
  label: string;                         // reflects
  icon: string;                          // reflects
  vertical: boolean;                     // reflects, default: false
}

// Slots
// default — accepts bim-button or bim-toolbar-group elements
```

### bim-toolbar-group

```typescript
// Custom element: <bim-toolbar-group>
interface BimToolbarGroup extends LitElement {
  vertical: boolean;                     // reflects, default: false
}

// Slots
// default — accepts bim-button elements
```

### bim-viewport

```typescript
// Custom element: <bim-viewport>
interface BimViewport extends LitElement {
  // Wraps a canvas/container element for the 3D renderer.
  // Pass this element as the container to SimpleRenderer
  // or PostproductionRenderer.
}
```

### bim-tabs / bim-tab

```typescript
// Custom element: <bim-tabs>
interface BimTabs extends LitElement {
  // Manages active tab state.
}

// Custom element: <bim-tab>
interface BimTab extends LitElement {
  label: string;                         // reflects
  icon: string;                          // reflects
}
```

---

## Interactive Components

### bim-button

```typescript
// Custom element: <bim-button>
interface BimButton extends LitElement {
  label: string;                         // reflects
  icon: string;                          // reflects
  disabled: boolean;                     // reflects, default: false
  active: boolean;                       // reflects, default: false
  vertical: boolean;                     // reflects, default: false
  tooltipTitle: string;                  // reflects
  tooltipText: string;                   // reflects
}

// Events
"click"  // standard MouseEvent
```

### bim-checkbox

```typescript
// Custom element: <bim-checkbox>
interface BimCheckbox extends LitElement {
  label: string;                         // reflects
  checked: boolean;                      // reflects, default: false
  inverted: boolean;                     // reflects, default: false
}

// Events
"change"  // checked state toggled
```

### bim-text-input

```typescript
// Custom element: <bim-text-input>
interface BimTextInput extends LitElement {
  label: string;                         // reflects
  value: string;
  placeholder: string;                   // reflects
  vertical: boolean;                     // reflects, default: false
}

// Events
"input"   // value changed during typing
"change"  // value committed
```

### bim-number-input

```typescript
// Custom element: <bim-number-input>
interface BimNumberInput extends LitElement {
  label: string;                         // reflects
  value: number;
  min: number;                           // reflects
  max: number;                           // reflects
  step: number;                          // reflects
  suffix: string;                        // reflects (unit label, e.g., "m")
  vertical: boolean;                     // reflects, default: false
  slider: boolean;                       // reflects — show as slider
  pref: string;                          // prefix text
}

// Events
"change"  // value changed
```

### bim-color-input

```typescript
// Custom element: <bim-color-input>
interface BimColorInput extends LitElement {
  label: string;                         // reflects
  color: string;                         // hex color value
  opacity: number;                       // 0-1
}

// Events
"change"  // color or opacity changed
```

### bim-dropdown

```typescript
// Custom element: <bim-dropdown>
interface BimDropdown extends LitElement {
  label: string;                         // reflects
  multiple: boolean;                     // reflects, default: false
  required: boolean;                     // reflects, default: false
  visible: boolean;                      // reflects — dropdown open state
  value: Record<string, string>[];       // selected options
}

// Events
"change"  // selection changed

// Slots
// default — accepts bim-option elements
```

### bim-option

```typescript
// Custom element: <bim-option>
interface BimOption extends LitElement {
  label: string;                         // reflects
  value: string;                         // reflects
  checked: boolean;                      // reflects, default: false
  icon: string;                          // reflects
  img: string;                           // reflects
  checkbox: boolean;                     // reflects — show checkbox indicator
}
```

### bim-selector

```typescript
// Custom element: <bim-selector>
interface BimSelector extends LitElement {
  label: string;                         // reflects
  multiple: boolean;                     // reflects, default: false
}

// Events
"change"  // selection changed

// Slots
// default — accepts bim-option elements
```

---

## Display Components

### bim-label

```typescript
// Custom element: <bim-label>
interface BimLabel extends LitElement {
  icon: string;                          // reflects — Iconify icon name
  img: string;                           // reflects — image URL
}

// Slots
// default — text content
```

### bim-icon

```typescript
// Custom element: <bim-icon>
interface BimIcon extends LitElement {
  icon: string;                          // reflects — Iconify icon name
}
```

### bim-table

```typescript
// Custom element: <bim-table>
interface BimTable extends LitElement {
  /** Row data with optional hierarchy. */
  data: TableGroupData[];

  /** Column definitions or simple name strings. */
  columns: (ColumnData | string)[];

  /** Computed filtered/grouped data (readonly). */
  value: TableGroupData[];

  /** Hide column headers. */
  headersHidden: boolean;                // default: false

  /** Minimum column width. */
  minColWidth: string;                   // default: "4rem"

  /** Expand all grouped rows. */
  expanded: boolean;                     // default: false

  /** Show loading indicator. */
  loading: boolean;                      // default: false

  /** Remove row indentation for children. */
  noIndentation: boolean;                // default: false

  /** Hide expand/collapse carets. */
  noCarets: boolean;                     // default: false

  /** Enable row selection. */
  selectableRows: boolean;              // default: false

  /** Set of selected row data. */
  selection: DataSet<object>;

  /** Simple text or column query filter. */
  queryString: string;

  /** Columns to group by. */
  groupedBy: string | string[];

  /** Keep hierarchy when filtering. */
  preserveStructureOnFilter: boolean;    // default: false

  /** Custom filter function. */
  filterFunction: (data: any) => boolean;

  /** Transform functions per column. */
  dataTransform: Record<string, (v: any) => any>;

  /** Default column visibility. */
  defaultVisibility: boolean;            // default: true
  visibilityExceptions: string[];
  hiddenColumns: string[];               // setter
  visibleColumns: string[];              // setter

  /** Export table data. */
  downloadData(fileName: string, format: "csv" | "tsv"): void;
}

interface TableGroupData<T = Record<string, any>> {
  data: Partial<T>;
  children?: TableGroupData<T>[];
}

interface ColumnData {
  name: string;
  width?: string;                        // CSS grid size, default: "minmax(minColWidth, 1fr)"
  forceDataTransform?: boolean;
}

// Events
"columnschange"          // columns updated
"dataselected"           // row selected, detail: { data }
"datadeselected"         // row deselected, detail: { data }
"dataselectioncleared"   // all selection cleared
"connected"              // component mounted
"disconnected"           // component unmounted
```

### bim-tooltip

```typescript
// Custom element: <bim-tooltip>
interface BimTooltip extends LitElement {
  // Renders tooltip content on hover over parent element.
}
```

### bim-context-menu

```typescript
// Custom element: <bim-context-menu>
interface BimContextMenu extends LitElement {
  // Right-click context menu with positioned overlay.
}
```

### bim-chart / bim-chart-legend

```typescript
// Custom element: <bim-chart>
interface BimChart extends LitElement {
  // Data visualization chart component.
}

// Custom element: <bim-chart-legend>
interface BimChartLegend extends LitElement {
  // Legend for chart component.
}
```

---

## CSS Custom Properties Reference

All properties use the `--bim-ui_` prefix and can be overridden at any DOM level.

### Background Colors

| Property | Default (Dark) | Description |
|---|---|---|
| `--bim-ui_bg-base` | `#1a1a2e` | Primary background |
| `--bim-ui_bg-contrast-10` | `#232340` | Subtle contrast layer |
| `--bim-ui_bg-contrast-20` | `#2c2c4a` | Borders, dividers |
| `--bim-ui_bg-contrast-40` | `#3d3d5c` | Muted elements |
| `--bim-ui_bg-contrast-60` | `#5a5a7a` | Secondary text |
| `--bim-ui_bg-contrast-80` | `#8888a8` | Primary text |
| `--bim-ui_bg-contrast-100` | `#ffffff` | Maximum contrast |

### Accent Colors

| Property | Default | Description |
|---|---|---|
| `--bim-ui_accent-base` | `#6528d7` | Primary accent (buttons, links) |
| `--bim-ui_accent-contrast` | `#ffffff` | Text on accent backgrounds |

### Sizing Scale

| Property | Default | Description |
|---|---|---|
| `--bim-ui_size-base` | `0.25rem` | Base unit for spacing/radius |
| `--bim-ui_size-2xs` | `0.125rem` | Extra-extra-small |
| `--bim-ui_size-xs` | `0.25rem` | Extra-small |
| `--bim-ui_size-sm` | `0.75rem` | Small (label font-size) |
| `--bim-ui_size-md` | `1rem` | Medium |
| `--bim-ui_size-lg` | `1.25rem` | Large |
| `--bim-ui_size-xl` | `1.5rem` | Extra-large |

### Typography

| Property | Default | Description |
|---|---|---|
| `--bim-ui_font-family` | `"Inter", sans-serif` | Global font family |

### Component-Specific Overrides

```css
/* Override label styling on a specific element */
bim-label {
  --bim-label--c: #ff0000;    /* text color */
  --bim-label--fz: 14px;      /* font size */
}
```

---

## @thatopen/ui-obc Factory Functions

All ui-obc components are factory functions returning `[HTMLElement, UpdateFunction]`.

### Tables

```typescript
// Spatial structure tree
CUI.tables.spatialTree(config: {
  components: OBC.Components;
  models: OBC.FragmentsModel[];
}): [HTMLElement, (config: Partial<typeof config>) => void];

// Element property data
CUI.tables.itemsData(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// Loaded models list
CUI.tables.modelsList(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// BCF viewpoints table
CUI.tables.viewpointsList(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// BCF topics table
CUI.tables.topicsList(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// BCF comments table
CUI.tables.commentsList(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];
```

### Buttons

```typescript
// IFC file load button
CUI.buttons.loadIfc(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// Fragment file load button
CUI.buttons.loadFrag(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];
```

### Sections

```typescript
// BCF topic information panel
CUI.sections.topicInformation(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// BCF topic comments panel
CUI.sections.topicComments(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// BCF topic relations panel
CUI.sections.topicRelations(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];

// BCF topic viewpoints panel
CUI.sections.topicViewpoints(config: {
  components: OBC.Components;
}): [HTMLElement, (config: Partial<typeof config>) => void];
```
