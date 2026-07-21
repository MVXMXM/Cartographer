# Rendering to HTML

The fallback renderer, used when no Figma integration is connected (or when the user prefers a file over a board). It produces the **same diagram** — same `color = WHO`, `shape = WHAT` visual system — as a single, self-contained `.html` document.

## Output contract

- **One file, written to disk.** Default path: `./<flow-name-kebab>.html` in the working directory (e.g. `acme-analytics-signup-flow.html`). Tell the user the exact path you wrote.
- **Fully self-contained.** All CSS inline in a single `<style>` block; the diagram drawn with HTML elements + one inline SVG layer for connectors. **No external requests** — no CDN, no Google Fonts `@import`, no `<script src>`. A title that needs a serif uses the system serif stack (`Georgia, 'Times New Roman', serif`); body text uses the system sans stack (`-apple-system, system-ui, sans-serif`). This keeps the file portable and openable offline.
- **No build step.** Opening the file in any browser shows the finished diagram.

## Visual system mapping

The palette and shape semantics are identical to FigJam — see [`palette.md`](palette.md) and [`shape-semantics.md`](shape-semantics.md). Reproduce them in CSS exactly:

```css
:root {
  /* entity fills — color = WHO (palette.md) */
  --human:  #C2E5FF;   /* light blue   */
  --ai:     #DCCCFF;   /* light violet */
  --system: #D9D9D9;   /* light gray   */
  /* outcome fills — valence, only on ellipses after a decision */
  --positive: #CDF4D3; /* light green */
  --negative: #FFCDC2; /* light red   */
  --ink:      #1E1E1E; /* all shape text */
  --note:     #B3B3B3; /* dashed note stroke */
  --connector:#757575; /* arrows */
}
```

**Stroke policy carries over:** filled shapes get no border. The only bordered shape is the Note (transparent fill, `2px dashed var(--note)`).

### Shapes (`shape = WHAT`)

Each node is a flex-centered box sized from its label (mirror `fitShape`'s grid: 1 line ≈ 112px tall, +32px per extra line, wrap text near ~240px). Shape is expressed with CSS:

| Kind | Shape | CSS |
|---|---|---|
| start | `CHEVRON` | `clip-path: polygon(0 0, 85% 0, 100% 50%, 85% 100%, 0 100%);` |
| state / action | `SQUARE` | plain rectangle, `border-radius: 0` |
| human action | `ROUNDED_RECTANGLE` | `border-radius: 16px` |
| decision | `DIAMOND` | a square with the **content held upright** inside a 45°-rotated background, or `clip-path: polygon(50% 0, 100% 50%, 50% 100%, 0 50%)` |
| outcome | `ELLIPSE` | `border-radius: 50%`, ~102×102, fill = positive/negative |
| note | `SQUARE` dashed | transparent fill, dashed gray border |

Candidate shapes (cylinder, parallelogram, document, etc.) map to the same `clip-path` equivalents only when the flow actually uses them — and then add a legend entry, same rule as FigJam.

## Layout

Reuse the FigJam layout model (see [`layout.md`](layout.md)) — the coordinates are computed the same way; only the drawing surface changes.

- **Container** is `position: relative`; every node is `position: absolute` at its computed `(x, y)`. This lets connectors line up exactly.
- **Left-to-right**, rows vertically center-aligned on a shared `rowCenterY`. Snake-wrap to a new row past ~1920px of width, alternating direction so the return connector is a short U-turn.
- **Branches** off a decision fan out as a vertical column; normalize the column to one shared width (the tight-width exception) so stacked nodes share a clean vertical edge.
- **Title** at the top (serif, large, with the ` (MM.DD.YYYY)` timestamp in regular weight), then the **legend** below it — only the entities the flow actually uses, as small filled dots with labels. **No outcome legend.**

## Connectors

Draw all arrows in **one inline `<svg>`** that overlays the container (`position:absolute; inset:0; pointer-events:none;`), sized to the full diagram. Use a shared arrowhead marker:

```html
<svg width="{W}" height="{H}">
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3"
            orient="auto" markerUnits="userSpaceOnUse">
      <path d="M0,0 L8,3 L0,6 Z" fill="#757575"/>
    </marker>
  </defs>
  <!-- straight when colinear: from right-center to left-center -->
  <line x1=".." y1=".." x2=".." y2=".." stroke="#757575" stroke-width="2" marker-end="url(#arrow)"/>
  <!-- elbowed for real corners (fan-out / snake return): use <polyline> -->
  <polyline points=".. .., .. .., .. .." fill="none" stroke="#757575" stroke-width="2" marker-end="url(#arrow)"/>
</svg>
```

Same rule as FigJam: a straight connector for colinear shapes (matched perpendicular centers), an elbow only for genuine direction changes. If a "straight" line kinks, fix the node placement, not the line. Unlabeled by default; labels (lowercased condition phrases) are opt-in via `connectorLabels: true`, rendered as small `<text>` at the segment midpoint.

## Validation (before writing the file)

Same flow-model invariants as FigJam — catch these before emitting markup:

- **A `DIAMOND` must have ≥ 2 outgoing connectors.** One-branch diamond → add the branch or downgrade to SQUARE/ROUNDED_RECTANGLE.
- An `ELLIPSE` outcome must sit downstream of a decision; a `CHEVRON` start has no incoming connector.

## Verification (after writing)

Re-read the generated file and confirm: title and timestamp present; legend lists exactly the entities used; no node label overflows its box; connector endpoints reference the intended node coordinates. If a frontend/browser screenshot tool is available, open the file and screenshot it to check for clipped text or overlaps — otherwise validate structurally. Fix in place rather than regenerating wholesale.
