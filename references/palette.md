# Palette

The core insight of this system: **color = WHO, shape = WHAT.** Color tells the reader which entity is responsible for a step; shape tells them what kind of step it is. The two compose freely — a "Square colored blue" means a Human-owned action; the same square in violet means an AI-owned action.

This separation is what makes the diagrams legible at a glance and extensible without inventing new shapes for every new entity.

## Stroke policy

**Filled shapes get no stroke.** The fill alone defines the silhouette; an outline on a filled shape adds visual noise without information. When rendering, set `shape.strokes = []` for every shape that has an entity or outcome fill.

The only exception is the **Note** shape, which has no fill — its dashed stroke *is* the entire visual.

Legend dots follow the same rule: filled with the entity color, no stroke.

## Entity colors

Use the FigJam palette directly — these hex values match FigJam's named swatches exactly so the rendered shapes look native, not custom.

| Entity | Fill | Text |
|---|---|---|
| Human | `#C2E5FF` (light blue) | `#1E1E1E` |
| AI Model | `#DCCCFF` (light violet) | `#1E1E1E` |
| Deterministic System | `#D9D9D9` (light gray) | `#1E1E1E` |

The legend at the top of every diagram displays these three by default, as small filled ellipses with the entity name to their right. If a flow uses only a subset of entities, omit the unused ones from the legend.

## Outcome colors

Outcomes (the leaves of a decision branch) are colored by valence, not by entity. They use the ELLIPSE shape (see [shape-semantics](shape-semantics.md)).

| Outcome | Fill | Text |
|---|---|---|
| Positive | `#CDF4D3` (light green) | `#1E1E1E` |
| Negative | `#FFCDC2` (light red) | `#1E1E1E` |

Use outcome colors *only* on ellipses, *only* immediately after a decision diamond. They should never carry an entity color — the green/red is the meaning.

## Annotation color

Notes and asides use a transparent fill with a **dashed gray stroke** to visually de-emphasize them.

| Element | Fill | Stroke | Stroke style |
|---|---|---|---|
| Note | none / `#FFFFFF` | `#B3B3B3` | dashed (`[10, 5]`) |

## Implementation notes

- **Filled shapes**: set `fill` and `text` only. Always explicitly set `shape.strokes = []` to clear FigJam's default stroke — otherwise a thin default stroke will sneak through.
- **Note shape**: no fill (transparent), only a dashed gray stroke. The stroke is the entire visual.
- **Use `hex/255` notation in code**, not rounded decimals — FigJam treats slightly-off values as "custom" colors and they stop matching the palette swatches in the UI.
- **Don't introduce additional entity colors** without updating this doc and the legend. New entity types (e.g., "Third-party service," "Background worker") should reuse the three existing colors and be distinguished by shape or a labeled note instead, unless the new entity genuinely earns its own swatch.

## Color reference snippet

```js
const h = (r, g, b) => ({ r: r / 255, g: g / 255, b: b / 255 });
const DARK = h(0x1E, 0x1E, 0x1E);

const ENTITY = {
  human:  { fill: h(0xC2, 0xE5, 0xFF), text: DARK },
  ai:     { fill: h(0xDC, 0xCC, 0xFF), text: DARK },
  system: { fill: h(0xD9, 0xD9, 0xD9), text: DARK },
};

const OUTCOME = {
  positive: { fill: h(0xCD, 0xF4, 0xD3), text: DARK },
  negative: { fill: h(0xFF, 0xCD, 0xC2), text: DARK },
};

// Note is the only shape with a stroke instead of a fill.
const NOTE = { stroke: h(0xB3, 0xB3, 0xB3), text: DARK, dash: [10, 5] };

// Apply to a filled shape:
function applyEntityPreset(shape, preset) {
  shape.fills = [{ type: 'SOLID', color: preset.fill }];
  shape.strokes = [];                                    // explicit — no outline
  shape.text.fills = [{ type: 'SOLID', color: preset.text }];
}

// Apply to a Note shape:
function applyNotePreset(shape) {
  shape.fills = [];                                      // no fill
  shape.strokes = [{ type: 'SOLID', color: NOTE.stroke }];
  shape.dashPattern = NOTE.dash;
  shape.text.fills = [{ type: 'SOLID', color: NOTE.text }];
}
```
