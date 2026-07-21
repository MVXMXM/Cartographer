# Rendering to FigJam

How the skill actually writes a diagram into a FigJam board via the official Figma plugin MCP (`use_figma` tool).

## Prerequisites

Verify in this order, and stop with a clear instruction if any check fails — never silently fall through.

1. **`use_figma` tool is available.** This is provided by the `figma@claude-plugins-official` plugin. If unavailable, tell the user:
   > FigJam output requires the official Figma plugin. Install it with:
   > ```
   > /plugin install figma@claude-plugins-official
   > ```
   > After install, re-authenticate when prompted (browser OAuth), then re-run me.

2. **A FigJam file URL is provided.** Format: `https://www.figma.com/board/:fileKey/:fileName?node-id=:nodeId`. Extract `fileKey` and `nodeId` from the URL. If the user hasn't given one, ask — do not create a new file on their behalf without explicit consent.

3. **The authenticated Figma account has view permission on the target file.** A `User does not have view permission on file` error means the OAuth account is wrong for this file. Surface the error and ask the user to either share the file with that account or re-authenticate with the right one.

## Render pipeline

Each render call follows the same five phases. Treat each phase as a separate `use_figma` invocation when possible — atomic, small steps make failures easy to recover from.

### 1. Inspect the target page

Before drawing, scan the page to find a clear area:

```js
const page = await figma.getNodeByIdAsync(pageNodeId);
await figma.setCurrentPageAsync(page);

let rightmostX = 0;
for (const child of page.children) {
  const r = child.x + child.width;
  if (r > rightmostX) rightmostX = r;
}
const originX = rightmostX === 0 ? 0 : rightmostX + 400;
return { originX };
```

Use `originX` as the X origin for the new diagram.

### 2. Load the title font

The FigJam title uses **`Merriweather`** — the typeface behind FigJam's "Bookish" style preset. Merriweather ships with FigJam by default, so no fallback is needed in this path. Load both Bold and Regular faces since the title is a mixed-weight run (Bold for the flow name, Regular for the timestamp).

```js
const titleBold    = { family: 'Merriweather', style: 'Bold' };
const titleRegular = { family: 'Merriweather', style: 'Regular' };
try {
  await figma.loadFontAsync(titleBold);
  await figma.loadFontAsync(titleRegular);
} catch (e) {
  // Fail loudly. Do NOT silently fall through to Inter or another Google Font
  // (Google Fonts share a delivery channel with Merriweather — if one fails, they all do).
  throw new Error(
    'Could not load Merriweather (the font behind FigJam "Bookish"). ' +
    'In Figma\'s font picker, search "Bookish" or "Merriweather" — if it doesn\'t appear, ' +
    'install Merriweather (free, SIL OFL) from Google Fonts and reload Figma.'
  );
}
```

### 3. Create the title and legend, defer row placement

This step **must** run before any flow shapes are placed — but the flow-shapes' `y` coordinates can't be set until we know how tall row 1 ends up. So the order is:

1. **Title**: single text node at `(originX, originY)` (where `originY` is the title's top edge), size `64`, characters `<flow name> (MM.DD.YYYY)`. Apply a **styled range**:
   - `setRangeFontName(0, nameEnd, titleBold)` for the flow name
   - `setRangeFontName(nameEnd, end, titleRegular)` for the ` (MM.DD.YYYY)` suffix (note the leading space is part of the regular run)
2. **Legend**: top edge at `legendY = title.y + title.height + 48`. Build it in two pieces, then **group them**:
   - Create alternating filled 16×16 ellipses (one per entity used in the flow) and a **single** mixed-weight text node containing the legend's full string (e.g. `Entity:  Human    AI Model    Deterministic System`). Apply `setRangeFontName` so:
     - `Entity:` is `{family: 'Inter', style: 'Bold'}`
     - Each entity name (`Human`, `AI Model`, `Deterministic System`) is `{family: 'Inter', style: 'Medium'}`
     - Load both Inter Bold and Inter Medium before setting characters.
   - Wrap them in a FigJam Group: `const legendGroup = figma.group(legendNodes, figma.currentPage); legendGroup.name = 'Legend';`
   - Capture `legendBottomY = legendGroup.y + legendGroup.height`.
   
   Include **only** entities the flow actually uses. **Do not render an outcome legend** — positive/negative ellipses are self-explanatory and an outcome legend is forbidden noise.

3. **Defer row-1 placement.** Do not place any flow shapes yet. Proceed to step 4 — `fitShape()` every shape that will be in row 1 first, find `row1MaxHeight`, then compute:

   ```js
   const row1TopY    = legendBottomY + 80;                    // 80 px is the strict minimum gap
   const row1CenterY = row1TopY + row1MaxHeight / 2;
   ```

   Every shape in row 1 is positioned at `shape.y = row1CenterY - shape.height / 2`. Subsequent rows derive their `rowCenterY` from the previous row's center plus `(prevRowMaxHeight + nextRowMaxHeight) / 2 + 160`.

   This is the only way to guarantee the legend never overlaps the tallest shape — the row-1 center moves with the content, the gap stays constant.

### 4. Create shapes

For every node in the synthesized flow model:

1. `figma.createShapeWithText()`.
2. Set `shapeType` (from [shape-semantics](shape-semantics.md)).
3. Set `text.characters` to the shape's label.
4. Size via `fitShape(label, shapeType)` (utility below) — **height comes from the line-count grid, width is tight to the text**. Never hardcode dimensions. Then, before placement, run `normalizeColumn()` on any vertically-stacked column (fan-out branches, stacked outcomes) so the column shares one width — the sole exception to tight width.
5. Apply the entity / outcome preset (see [palette](palette.md)). For filled shapes set `fills` and `text.fills`, then **explicitly clear `strokes = []`** — filled shapes have no outline.
6. For Note shapes only: no fill, dashed gray stroke (`strokes = [...]` with `dashPattern = [10, 5]`).
7. Position based on the layout rules (see [layout](layout.md)). Shapes in a row are **vertically center-aligned** to a shared `rowCenterY`, not top-aligned. Compute `shape.y = rowCenterY - shape.height / 2`.
8. Track the row's running right edge as you place shapes. When the next shape would exceed `originX + 1920 px`, **snake** to a new row — start the new row at `originX`, increment `rowCenterY` by `prevRowHeight + 160 px`, and alternate the row direction so the snake connector is a short U-turn.

**No background shape and no Section wrapper.** The diagram sits directly on the FigJam grid. Do not create a rectangle behind the flow as a backdrop and do not wrap the flow in a FigJam Section. The grid is the canvas; the flow is the content.

Always preload `{family: 'Inter', style: 'Medium'}` (the ShapeWithText default font) once at the start of the call.

#### Shape-sizing utility

The utility locks height to the line-count grid (same grid for every rectangular shape type) and makes width tight to the text — only as wide as the longest wrapped line. Shape-specific width padding compensates for FigJam's internal text-area shrinkage on shapes with rounded corners or non-rectangular geometry.

```js
const HEIGHT_BY_LINES = { 1: 112, 2: 144, 3: 176, 4: 208 };
const MAX_LINES_KEY = 4;

const WIDTH_PAD_BY_SHAPE = {
  SQUARE:               0,
  ROUNDED_RECTANGLE:    30,
  PARALLELOGRAM_RIGHT:  30,
  PARALLELOGRAM_LEFT:   30,
  TRAPEZOID:            40,
  CHEVRON:              50,
};

const INTRINSIC_SIZE = {
  DIAMOND: { width: 192, height: 184 },
  ELLIPSE: { width: 102, height: 102 },
};

const INNER_PAD = 32;        // horizontal padding inside the shape
const LINE_HEIGHT_PX = 22;   // Inter Medium 16px line height in FigJam ShapeWithText

// Single measurer text node reused across calls; create once, remove at end.
async function makeMeasurer() {
  const m = figma.createText();
  m.fontName = { family: 'Inter', style: 'Medium' };
  m.textAutoResize = 'HEIGHT';
  return m;
}

const MAX_TEXT_WIDTH = 240;  // wrap ceiling — text never gets wider than this before wrapping

function fitShape(label, shapeType, measurer) {
  // Exempt shapes use their intrinsic geometry — don't apply the grid or width pad.
  if (INTRINSIC_SIZE[shapeType]) return { ...INTRINSIC_SIZE[shapeType] };

  // 1. Measure the natural single-line width (no wrapping).
  measurer.textAutoResize = 'WIDTH_AND_HEIGHT';
  measurer.characters = label;
  const naturalWidth = measurer.width;

  let textWidth, lines;
  if (naturalWidth <= MAX_TEXT_WIDTH) {
    // Fits on one line — tight width is the natural width, no wrapping needed.
    textWidth = naturalWidth;
    lines = 1;
  } else {
    // Exceeds the ceiling — wrap at MAX_TEXT_WIDTH and count lines.
    measurer.textAutoResize = 'HEIGHT';
    measurer.resize(MAX_TEXT_WIDTH, measurer.height);
    measurer.characters = label;
    lines = Math.min(Math.max(1, Math.ceil(measurer.height / LINE_HEIGHT_PX)), MAX_LINES_KEY);

    // Tighten: shrink the width to the longest *actual* wrapped line, keeping the
    // same line count. A wrapped block rarely fills the full MAX_TEXT_WIDTH — this
    // strips the dead horizontal space so the shape is as narrow as the text allows.
    let lo = 0, hi = MAX_TEXT_WIDTH;
    while (hi - lo > 4) {
      const mid = (lo + hi) / 2;
      measurer.resize(mid, measurer.height);
      const l = Math.max(1, Math.ceil(measurer.height / LINE_HEIGHT_PX));
      if (l <= lines) hi = mid; else lo = mid;
    }
    textWidth = hi;
  }

  // 2. Snap height to the line-count grid.
  const height = HEIGHT_BY_LINES[lines];

  // 3. Compute width: tight measured text block + interior padding + per-shape pad.
  const widthPad = WIDTH_PAD_BY_SHAPE[shapeType] ?? 0;
  const width = Math.round(textWidth + INNER_PAD * 2 + widthPad);

  return { width, height };
}
```

Two shapes with the same label and the same line count produce the same height regardless of shape type. Their **widths differ** by the actual length of their text — width is tight to the longest wrapped line, never a shared target. The one place widths are forced equal is column normalization (next).

#### Column normalization

Tight width is the default; the exception is shapes locked into a vertical column (fan-out branches off a diamond, stacked outcomes). Run this **after** `fitShape()` has sized every shape and **before** final placement. See [layout](layout.md#column-normalization-the-tight-width-exception) for the rule.

```js
// `column` = an array of shape records ({ shapeType, width, height, columnCenterX, ... })
// that share one flow position and stack vertically. Build these groups from the flow
// model (a diamond's fan-out branches; a stack of outcomes), not from raw x proximity.
function normalizeColumn(column) {
  // Only the rectangular family is stretched — diamonds/ellipses keep intrinsic width.
  const stretchable = column.filter(s => !INTRINSIC_SIZE[s.shapeType]);
  if (stretchable.length < 2) return;            // nothing to align

  const colWidth = Math.max(...stretchable.map(s => s.width));
  for (const s of stretchable) {
    s.width = colWidth;
    s.x = s.columnCenterX - colWidth / 2;        // recenter on the column's flow axis
  }
}
```

### 5. Validate the flow model

Before creating any connectors, validate the synthesized flow against shape-semantic invariants. Catching these here is much cheaper than rendering a wrong diagram and fixing it post-hoc.

**DIAMOND must have ≥ 2 outgoing connectors.** A decision with one path is not a decision. For every node where `kind === 'decision'`, count the outgoing edges in your flow model. If any diamond has fewer than 2, **stop and ask the user**:

> The decision point "<label>" has only one outgoing path. A decision needs ≥ 2 branches.
> Options:
> 1. Add the missing branch (what's the other outcome?)
> 2. Downgrade this node from a decision to a regular step (SQUARE or ROUNDED_RECTANGLE).
> Which do you prefer?

Do not silently render a one-branch diamond — it misleads the reader into expecting a branch that isn't there.

Apply analogous validations as the model grows: e.g. an ELLIPSE (outcome) must be downstream of a DIAMOND; a CHEVRON (start) must have no incoming connectors. Surface violations before rendering.

### 6. Create connectors

Pick the magnets and line type from the **geometric relationship** between the two shapes — this is what keeps connectors from kinking (see the "Connectors must be colinear" rule in [layout](layout.md)).

```js
function connect(fromShape, toShape, { branch = false } = {}) {
  const c = figma.createConnector();

  const fromCenterY = fromShape.y + fromShape.height / 2;
  const toCenterY   = toShape.y   + toShape.height / 2;
  const fromCenterX = fromShape.x + fromShape.width / 2;
  const toCenterX   = toShape.x   + toShape.width / 2;

  const colinearHorizontal = Math.abs(fromCenterY - toCenterY) < 0.5 && toCenterX > fromCenterX;
  const colinearVertical   = Math.abs(fromCenterX - toCenterX) < 0.5 && toCenterY > fromCenterY;

  if (colinearHorizontal) {
    // Same axis, left-to-right: explicit side magnets → perfectly straight, no kink.
    c.connectorStart = { endpointNodeId: fromShape.id, magnet: 'RIGHT' };
    c.connectorEnd   = { endpointNodeId: toShape.id,   magnet: 'LEFT'  };
    c.connectorLineType = 'STRAIGHT';
  } else if (colinearVertical) {
    // Same axis, top-to-bottom: straight vertical drop.
    c.connectorStart = { endpointNodeId: fromShape.id, magnet: 'BOTTOM' };
    c.connectorEnd   = { endpointNodeId: toShape.id,   magnet: 'TOP'    };
    c.connectorLineType = 'STRAIGHT';
  } else {
    // Genuine direction change (fan-out branch, snake-wrap return): a corner is intended.
    c.connectorStart = { endpointNodeId: fromShape.id, magnet: branch ? 'BOTTOM' : 'RIGHT' };
    c.connectorEnd   = { endpointNodeId: toShape.id,   magnet: 'AUTO' };
    c.connectorLineType = 'ELBOWED';
  }

  c.connectorStartStrokeCap = 'NONE';
  c.connectorEndStrokeCap = 'ARROW_LINES';
  c.strokes = [{ type: 'SOLID', color: h(0x75, 0x75, 0x75) }]; // FigJam dark gray
  c.strokeWeight = 2;
  // no label by default
  return c;
}
```

**The kink is a misalignment symptom, not a line-type problem.** If a connector that should be straight comes out elbowed, the two shapes' perpendicular-axis centers don't match — fix the *placement* (copy the predecessor's center-x for a vertical drop; rely on the shared `rowCenterY` for a horizontal hop) rather than forcing the line type. A straight connector between truly-colinear shapes never kinks.

ELBOWED is correct only for real corners: a decision fanning out to an outcome that's offset on both axes, or a snake-wrap returning to the next row.

## Connector labels (opt-in)

The default is unlabeled. To enable labels, the caller passes `connectorLabels: true` to the skill. When enabled:

```js
const font = { family: 'Inter', style: 'Medium' };
await figma.loadFontAsync(font);
// for each connector that has a label:
c.text.fontName = font;            // MUST set fontName before characters
c.text.characters = label;
```

Labels are lowercased trigger / condition phrases ("yes," "no," "payment added," "key created"). Never use them when the shape's color or content already says the same thing — they should add information, not echo it.

## Atomic recovery

If a `use_figma` call fails partway through, no changes are written (each script is atomic). Read the error, fix, and retry. Common failures:
- `User does not have view permission on file` — the authenticated Figma account lacks access. Ask the user to share the file or re-auth with the correct account.
- Font not found — the chosen `titleFont` isn't installed; fall back to the next candidate or `Inter Bold`.
- `Cannot read properties of null` — a node ID from a prior call no longer exists. Re-fetch the page contents and retry.

Always return created node IDs from every `use_figma` call so subsequent calls can reference them.

## Verification

After rendering, call `get_screenshot` on the page node and visually verify:
- Title is readable and not clipped.
- Legend correctly enumerates the entities used in the flow.
- No shape text is truncated (if it is, raise `MAX_TEXT_WIDTH` or the `MAX_LINES_KEY` cap in `fitShape`).
- Each shape is only as wide as its text needs — except columns of stacked shapes, which share one width and line up on a clean vertical edge.
- Connectors are attached on the expected sides (no crossings that obscure other connectors).
- The diagram is positioned clear of pre-existing content.

If any check fails, write a targeted fix script rather than recreating the whole diagram.
