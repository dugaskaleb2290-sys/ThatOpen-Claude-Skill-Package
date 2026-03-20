# Anti-Patterns — UI Component Syntax

Version: @thatopen/ui 3.3.x / @thatopen/ui-obc 3.3.x

Common mistakes when using ThatOpen UI components — and what to do instead.

---

## AP-001: Missing BUI.Manager.init()

**WRONG:**
```typescript
import * as BUI from "@thatopen/ui";

// Immediately create bim-* elements without init
const panel = document.createElement("bim-panel");
panel.name = "Properties";
document.body.appendChild(panel);
// Result: <bim-panel> renders as unknown element — no styling, no behavior
```

**CORRECT:**
```typescript
import * as BUI from "@thatopen/ui";

// ALWAYS call init() before any bim-* element usage
BUI.Manager.init();

const panel = document.createElement("bim-panel");
panel.name = "Properties";
document.body.appendChild(panel);
```

**Why:** `BUI.Manager.init()` registers all `bim-*` custom elements with
the browser's Custom Elements Registry and injects global styles. Without it,
the browser treats `bim-*` tags as generic `HTMLElement` instances — they
render empty with no Shadow DOM, no styles, and no reactive properties. The
failure is silent: no errors appear in the console.

---

## AP-002: Wrong Element Tag Names

**WRONG:**
```typescript
// These element names do NOT exist in @thatopen/ui
const panel = document.createElement("bim-side-panel");      // wrong
const section = document.createElement("bim-section");        // wrong
const input = document.createElement("bim-input-text");       // wrong
const group = document.createElement("bim-button-group");     // wrong
```

**CORRECT:**
```typescript
// Use the EXACT registered element names
const panel = document.createElement("bim-panel");
const section = document.createElement("bim-panel-section");
const input = document.createElement("bim-text-input");
const group = document.createElement("bim-toolbar-group");
```

**Why:** Custom elements MUST match the exact tag name registered by
`BUI.Manager.init()`. Misspelled or invented names create generic HTML
elements with no functionality. There is no fuzzy matching or helpful
error message — the browser silently creates an unknown element.

### Complete Tag Name Reference

| Correct Name | Common Mistakes |
|---|---|
| `bim-panel` | `bim-side-panel`, `bim-sidebar` |
| `bim-panel-section` | `bim-section`, `bim-panel-group` |
| `bim-toolbar` | `bim-menubar`, `bim-actionbar` |
| `bim-toolbar-section` | `bim-toolbar-group` (this is a different element) |
| `bim-toolbar-group` | `bim-button-group`, `bim-group` |
| `bim-text-input` | `bim-input-text`, `bim-input`, `bim-textfield` |
| `bim-number-input` | `bim-input-number`, `bim-numeric-input` |
| `bim-color-input` | `bim-input-color`, `bim-colorpicker` |
| `bim-dropdown` | `bim-select`, `bim-combobox` |
| `bim-checkbox` | `bim-toggle`, `bim-switch` |

---

## AP-003: Setting Grid Layout Before Defining Layouts

**WRONG:**
```typescript
const grid = document.createElement("bim-grid");

// Setting layout name before layouts are defined — logs warning, nothing renders
grid.layout = "main";

grid.layouts = {
  main: {
    template: `"viewport" 1fr / 1fr`,
    elements: { viewport },
  },
};
```

**CORRECT:**
```typescript
const grid = document.createElement("bim-grid");

// ALWAYS define layouts first
grid.layouts = {
  main: {
    template: `"viewport" 1fr / 1fr`,
    elements: { viewport },
  },
};

// THEN activate
grid.layout = "main";
```

**Why:** When `layout` is set, the Grid component looks up the name in its
`layouts` collection. If the collection is empty or the name does not exist,
it logs a warning and renders nothing. The layout name MUST exist in the
layouts object at the time it is assigned.

---

## AP-004: Directly Styling Shadow DOM Internals

**WRONG:**
```css
/* These selectors CANNOT pierce Shadow DOM */
bim-panel .header { background: red; }
bim-button span { color: blue; }
bim-toolbar > div { gap: 10px; }
```

**CORRECT:**
```css
/* Use CSS custom properties — they DO cross Shadow DOM boundaries */
bim-panel {
  --bim-ui_bg-base: #1e293b;
  --bim-ui_bg-contrast-80: #e2e8f0;
}

bim-button {
  --bim-ui_accent-base: #3b82f6;
}
```

**Why:** `bim-*` components use Shadow DOM for style encapsulation. External
CSS selectors cannot target elements inside the Shadow Root. CSS custom
properties are the ONLY way to theme these components from outside, because
custom properties inherit through Shadow DOM boundaries by design.

---

## AP-005: Using bim-* Elements in HTML Without init()

**WRONG:**
```html
<!-- In static HTML without any JavaScript initialization -->
<bim-grid layout="main">
  <bim-toolbar>
    <bim-toolbar-section label="File">
      <bim-button label="Open" icon="folder"></bim-button>
    </bim-toolbar-section>
  </bim-toolbar>
</bim-grid>
```

**CORRECT:**
```html
<bim-grid layout="main">
  <bim-toolbar>
    <bim-toolbar-section label="File">
      <bim-button label="Open" icon="folder"></bim-button>
    </bim-toolbar-section>
  </bim-toolbar>
</bim-grid>

<script type="module">
  import * as BUI from "@thatopen/ui";
  // ALWAYS init — even when using declarative HTML
  BUI.Manager.init();
</script>
```

**Why:** HTML custom element tags are parsed by the browser but remain
unresolved until their constructors are registered via `customElements.define()`.
`BUI.Manager.init()` calls `customElements.define()` for every `bim-*` element.
Without the script, all elements exist in the DOM as undefined custom elements
with no rendering behavior.

---

## AP-006: Mixing Up @thatopen/ui and @thatopen/ui-obc

**WRONG:**
```typescript
import * as BUI from "@thatopen/ui";

// ui-obc components are NOT part of @thatopen/ui
const [tree, update] = BUI.tables.spatialTree({ components });  // ERROR
const [btn, updateBtn] = BUI.buttons.loadIfc({ components });   // ERROR
```

**CORRECT:**
```typescript
import * as BUI from "@thatopen/ui";
import * as CUI from "@thatopen/ui-obc";

BUI.Manager.init();

// Functional BIM components come from ui-obc
const [tree, updateTree] = CUI.tables.spatialTree({ components, models: [] });
const [btn, updateBtn] = CUI.buttons.loadIfc({ components });
```

**Why:** `@thatopen/ui` provides **presentational** web components with no
engine dependency. `@thatopen/ui-obc` provides **functional** BIM components
that require `@thatopen/components`. They are separate packages with different
imports. Attempting to access ui-obc APIs from the ui import results in
`TypeError: Cannot read properties of undefined`.

---

## AP-007: Forgetting to Call the Update Function (ui-obc)

**WRONG:**
```typescript
const [tree, updateTree] = CUI.tables.spatialTree({ components, models: [] });

// Model loaded but tree is never updated — still shows empty
fragments.onFragmentsLoaded.add((model) => {
  // Missing updateTree call
});
```

**CORRECT:**
```typescript
const [tree, updateTree] = CUI.tables.spatialTree({ components, models: [] });

fragments.onFragmentsLoaded.add((model) => {
  // ALWAYS call the update function with new data
  updateTree({ models: [model] });
});
```

**Why:** ui-obc factory functions return `[element, updateFunction]`. The
element is created once with the initial config. When engine state changes
(new model loaded, selection changed), you MUST call the update function
to re-render the component with new data. The component does NOT
automatically observe engine state changes.

---

## AP-008: Nesting Components in Wrong Parents

**WRONG:**
```html
<!-- bim-option directly in body — must be inside dropdown or selector -->
<bim-option label="Concrete"></bim-option>

<!-- bim-panel-section outside a panel — loses form value aggregation -->
<div>
  <bim-panel-section label="Settings">
    <bim-text-input label="Name"></bim-text-input>
  </bim-panel-section>
</div>

<!-- bim-toolbar-section outside toolbar — loses vertical/label propagation -->
<div>
  <bim-toolbar-section label="Tools">
    <bim-button label="Measure"></bim-button>
  </bim-toolbar-section>
</div>
```

**CORRECT:**
```html
<bim-dropdown label="Material">
  <bim-option label="Concrete"></bim-option>
</bim-dropdown>

<bim-panel name="Settings">
  <bim-panel-section label="General">
    <bim-text-input label="Name"></bim-text-input>
  </bim-panel-section>
</bim-panel>

<bim-toolbar>
  <bim-toolbar-section label="Tools">
    <bim-button label="Measure"></bim-button>
  </bim-toolbar-section>
</bim-toolbar>
```

**Why:** ThatOpen UI components use slot-based composition. Parent components
propagate properties to children via `slotchange` events (e.g., toolbar sets
`vertical` and `labelsHidden` on its sections). Panels aggregate form values
from their sections. Placing child components outside their expected parent
breaks property propagation and event bubbling.

---

## AP-009: Setting Properties Before Custom Element Upgrade

**WRONG:**
```typescript
// Element created but init() not called yet
const btn = document.createElement("bim-button");
btn.label = "Open";        // Sets property on HTMLElement — lost after upgrade
btn.icon = "mdi:folder";   // Same issue

// Later...
BUI.Manager.init();        // Element upgrades, but properties were set on wrong prototype
```

**CORRECT:**
```typescript
// ALWAYS init first
BUI.Manager.init();

// Now properties are set on the correctly upgraded element
const btn = document.createElement("bim-button");
btn.label = "Open";
btn.icon = "mdi:folder";
```

**Alternative** (if init order cannot be guaranteed):
```typescript
await customElements.whenDefined("bim-button");
const btn = document.createElement("bim-button");
btn.label = "Open";
```

**Why:** Before `customElements.define()` is called (via `BUI.Manager.init()`),
`document.createElement("bim-button")` creates a generic `HTMLElement`.
Properties set on this generic element may not transfer correctly when the
element is later "upgraded" to the custom element class. This leads to
properties appearing to be set but not reflected in the rendered output.
