# Highlighting Anti-Patterns

## Missing setup() Call

**WRONG** — Using Highlighter without calling setup():

```typescript
const highlighter = components.get(OBCF.Highlighter);
// Missing: highlighter.setup({ world });
await highlighter.highlight("select"); // Fails — no world reference
```

**CORRECT** — ALWAYS call setup() first:

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });
await highlighter.highlight("select");
```

`setup()` initializes the world reference, creates the default style, and
binds click event listeners. Without it, all operations fail silently or
throw.

---

## Configuring Before setup()

**WRONG** — Setting properties before initialization:

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.multiple = "shiftKey";     // Too early
highlighter.autoToggle.add("select");  // Too early
highlighter.setup({ world });
```

**CORRECT** — Configure AFTER setup():

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });
highlighter.multiple = "shiftKey";
highlighter.autoToggle.add("select");
```

---

## Outliner Without PostproductionRenderer

**WRONG** — Using Outliner with SimpleRenderer:

```typescript
world.renderer = new OBC.SimpleRenderer(components, container);

const outliner = components.get(OBCF.Outliner);
outliner.styles.add("select");
// Outlines never render — SimpleRenderer has no post-processing pipeline
```

**CORRECT** — ALWAYS use PostproductionRenderer with Outliner:

```typescript
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.renderer.postproduction.enabled = true;

const outliner = components.get(OBCF.Outliner);
outliner.styles.add("select");
```

---

## Outliner Without Enabling Postproduction

**WRONG** — PostproductionRenderer but postproduction not enabled:

```typescript
world.renderer = new OBCF.PostproductionRenderer(components, container);
// Missing: world.renderer.postproduction.enabled = true;

const outliner = components.get(OBCF.Outliner);
outliner.styles.add("select");
// Outlines do not render — postproduction pipeline is disabled
```

**CORRECT**:

```typescript
world.renderer = new OBCF.PostproductionRenderer(components, container);
world.renderer.postproduction.enabled = true;

const outliner = components.get(OBCF.Outliner);
outliner.styles.add("select");
```

---

## Not Disposing Mesher Output

**WRONG** — Leaking meshes from Mesher:

```typescript
const meshes = mesher.get(items);
world.scene.three.add(...meshes);
// Later: meshes are never disposed → memory leak
```

**CORRECT** — ALWAYS dispose Mesher output manually:

```typescript
const meshes = mesher.get(items);
world.scene.three.add(...meshes);

// When done:
for (const mesh of meshes) {
  mesh.geometry.dispose();
  if (Array.isArray(mesh.material)) {
    mesh.material.forEach((m) => m.dispose());
  } else {
    mesh.material.dispose();
  }
  world.scene.three.remove(mesh);
}
```

Mesher returns standard THREE.Mesh objects that are NOT tracked by the
fragment system. The caller owns their lifecycle.

---

## Style Name Mismatch Between Highlighter and Outliner

**WRONG** — Outliner tracks a style name that does not exist in Highlighter:

```typescript
highlighter.styles.set("selected", { color, opacity: 0.5 });
outliner.styles.add("select"); // "select" ≠ "selected"
// Outlines never appear because the style name does not match
```

**CORRECT** — Use the exact same style name:

```typescript
highlighter.styles.set("select", { color, opacity: 0.5 });
outliner.styles.add("select");
```

---

## Confusing Hoverer and Highlighter

**WRONG** — Using Hoverer for selection state:

```typescript
hoverer.onHoverStarted.add((data) => {
  // Treating hover as selection — wrong
  selectedElements = data;
  updatePropertiesPanel(selectedElements);
});
```

**CORRECT** — Use Highlighter for selection, Hoverer for visual feedback:

```typescript
// Hoverer: visual feedback only
hoverer.onHoverStarted.add(() => {
  container.style.cursor = "pointer";
});

// Highlighter: actual selection state
await highlighter.highlight("select");
const selected = highlighter.selection["select"];
updatePropertiesPanel(selected);
```

---

## Using FastModelPicker Unnecessarily

**WRONG** — Using FastModelPicker for a small model:

```typescript
// Model has 500 elements — raycasting is fast enough
const picker = components.get(OBC.FastModelPicker);
// Unnecessary complexity and GPU overhead
```

**CORRECT** — Use the default Highlighter raycasting for most models:

```typescript
const highlighter = components.get(OBCF.Highlighter);
highlighter.setup({ world });
// Built-in raycasting handles models up to ~100k elements efficiently
```

Only switch to FastModelPicker when profiling shows raycasting is a
bottleneck with very large models (100k+ elements).

---

## Performance: Too Many Highlight Styles

**WRONG** — Creating hundreds of individual styles:

```typescript
// One style per element — extremely inefficient
for (const elementId of allElements) {
  highlighter.styles.set(`style-${elementId}`, {
    color: getColorForElement(elementId),
    opacity: 0.5,
  });
}
```

**CORRECT** — Group elements by color into shared styles:

```typescript
// Group by color — efficient
const colorGroups = groupElementsByColor(allElements);
for (const [color, elements] of colorGroups) {
  const styleName = `group-${color}`;
  highlighter.styles.set(styleName, {
    color: new THREE.Color(color),
    opacity: 0.5,
  });
  await highlighter.highlightByID(styleName, elements, false);
}
```

Each style triggers a separate render pass. Minimize the number of active
styles for better performance.

---

## Forgetting to Clear Previous Highlights

**WRONG** — Accumulating highlights without clearing:

```typescript
// Every click adds more highlights, never clearing
container.addEventListener("click", async () => {
  await highlighter.highlight("select", false); // removePrevious = false
});
// Selection grows unbounded, eventually consuming significant memory
```

**CORRECT** — Clear when appropriate:

```typescript
// Default behavior: removePrevious = true (clears before highlighting)
container.addEventListener("click", async () => {
  await highlighter.highlight("select"); // removePrevious defaults to true
});

// Or explicitly manage clearing:
async function resetAndHighlight(items: ModelIdMap) {
  await highlighter.clear("select");
  await highlighter.highlightByID("select", items);
}
```
