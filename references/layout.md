# Layout

The flow reads **left-to-right**, top to bottom for branches. Decisions fan outward; outcomes resolve cleanly.

## Spacing

| Measurement | Value | Why |
|---|---|---|
| Horizontal gap between shapes | **52 px** | Matches the reference; gives connectors room without feeling sparse. |
| Vertical gap between branch shapes | **52 px** (same as horizontal) | Consistent grid. |
| **Height grid (rectangular shapes)** | **`1 line: 112` · `2 lines: 144` · `3 lines: 176` · `4+ lines: 208`** | All rectangular-ish shapes (SQUARE, ROUNDED_RECTANGLE, CHEVRON, PARALLELOGRAM, TRAPEZOID) share this grid keyed on rendered line count. Height varies *only* with line count, never with shape type. |
| Diamond size | **192 × 184 px (intrinsic)** | Exempt from the height grid — diamonds have their own geometry tied to decision semantics. |
| Ellipse outcome size | **102 × 102 px (intrinsic)** | Exempt from the height grid — outcomes read as terminal nodes, not steps. |
| Title-to-legend gap | **48 px** (title bottom to legend top) | The title is large and needs visual breathing room. |
| **Legend-to-flow gap (minimum)** | **80 px** (legend bottom to *top edge of the tallest shape in row 1*) | Measured from the legend's bounding box to the top of the *tallest* shape in the first row — not from legend center to row center. This is the only sizing that prevents legend / shape overlap. |
| **Max row width** | **1920 px** | Past this, snake to the next row instead of growing horizontally. Tuned for typical design-spec canvas dimensions. |
| Row-to-row gap (when snaking) | **160 px** | Vertical breathing room between the snake-wrap rows. |

## Sizing rule

**Height is locked to the line-count grid. Width is tight to the text it wraps.**

Two shapes with the same line count must have the same height regardless of shape type. A 2-line ROUNDED_RECTANGLE and a 2-line SQUARE are both `144 px` tall. The reference Figma design system enforces this — every rectangular shape in it is `112 px` tall, full stop.

**Width takes up only as much space as the text needs.** A shape's width is the width of its *tightest* text block — the longest wrapped line — plus interior padding plus the shape-specific geometry pad below. A shape labeled "Sign in" is narrow; a shape labeled "Confirm your email address to continue" is wider. Do **not** size every shape to a common target width. Measuring tightly is the job of `fitShape()` (see [render-figjam](render-figjam.md)) — measure the natural single-line width first, and only wrap (clamping to the `240 px` text ceiling) when the label exceeds it; for a wrapped label, shrink the width back down to the longest actual line so the shape never carries dead horizontal space.

The lone exception is **column normalization** (below): shapes locked into a vertical column share a single width so the column reads as one aligned unit.

Width also compensates for shape-specific geometry. Rounded corners and chevron points reduce the usable interior text area, so those shapes get extra width to fit the same content at the same height:

| Shape | Width padding over SQUARE |
|---|---|
| SQUARE | 0 (baseline) |
| ROUNDED_RECTANGLE | +30 px |
| PARALLELOGRAM (left or right) | +30 px |
| TRAPEZOID | +40 px |
| CHEVRON | +50 px |

The width padding compensates for FigJam's internal text-area shrinkage — it isn't visual flair. Don't apply it to shapes outside the rectangular family.

DIAMOND and ELLIPSE are exempt from both the height grid and the width-padding rule — use their intrinsic sizes (above).

## Column normalization (the tight-width exception)

Tight per-shape width is the default. The one exception: **when two or more shapes are locked into the same vertical column, they all adopt the column's widest member's width.** This keeps the column reading as a single aligned block and keeps its content equidistant from the columns and connectors on either side. A column of three branch actions where one is wide and two are narrow looks ragged and ambiguous; normalizing them to a shared width makes the column a clean vertical edge.

A **column** is any set of shapes that share a single horizontal flow position and stack vertically — most commonly:
- the fan-out branches off a decision diamond (the actions arranged above/below the continuing path), and
- outcome ellipses or recovery shapes stacked at one flow step.

Shapes that merely happen to land at a similar `x` across *different* flow steps are **not** a column — normalization applies only to shapes that are deliberately stacked at one position.

Procedure, applied **after** `fitShape()` has sized every shape tightly and **before** final `x`/`y` placement:

1. Group shapes by column membership (same flow position, vertically stacked).
2. For each group of size ≥ 2, compute `colWidth = max(member.width)`.
3. Set every member's `width = colWidth`. Heights are untouched — they stay on the line-count grid.
4. Re-derive each member's `x` so the column stays centered on its flow axis: `member.x = columnCenterX - colWidth / 2`. Because every member now shares `colWidth`, their left and right edges line up exactly.

DIAMOND and ELLIPSE keep their intrinsic widths and are never stretched by column normalization — if a column mixes an ellipse with rectangular shapes, normalize only the rectangular family members among them. A single shape standing alone at a flow position is its own tight width; there is nothing to normalize.

## Direction

- **Default direction is horizontal** (left-to-right). The first shape (CHEVRON) sits at the left.
- **Snake on overflow.** Track the row's running width as you place shapes. If the next shape would push the row past **1920 px**, wrap to a new row below: the new row starts at the same `x` as the first shape, `y` increases by the previous row's height + 160 px row-gap. **Alternate row direction** — odd rows run left-to-right, even rows run right-to-left, so the snake connector is always a short U-turn rather than a long return.
- Draw the snake connector from the last shape of the previous row to the first shape of the next row with a single ELBOWED connector. Use `magnet: 'BOTTOM'` on the source and `magnet: 'TOP'` on the target so the connector visibly drops one row and turns.
- Branches off a decision diamond go **outward and downward** — to the right, to the left, or below, but **never upward**. See the "Title is the ceiling" rule below. The reference layout pattern:

```
... → DIAMOND ─→ NEGATIVE (right of diamond)
        │
        └─→ POSITIVE (below diamond) → continues flow
```

The "positive" / "happy path" outcome continues the flow; the "negative" / "off-path" outcome terminates or leads to a recovery branch.

## Branching from a non-binary decision

For a "User evaluates" diamond with multiple actions branching out (not a binary positive/negative), arrange the actions in a fan around the diamond:

```
              ACTION 1
                 ↑
... → DIAMOND ─→ ACTION 2 (continuing)
                 ↓
              ACTION 3
```

Each action gets its own connector from the diamond. Don't introduce a labeled connector unless the user has enabled connector labels — the action's name is the meaning.

## Title is the ceiling

The title's top edge is the **absolute ceiling** of the diagram. No shape, connector, or annotation may ever have a `y` coordinate that places its top edge above the title.

This means:
- **Branches always go down** (or sideways), never up. If a decision diamond would naturally fan one outcome upward, re-route it: place that branch to the right of the diamond at the same row, or drop it below alongside the other branches. Never above.
- **Notes attached to a row** sit either below the row or beside it on the same horizontal axis. They never float into the title/legend area.
- **Snake-wrap rows** always increase `y`. The diagram can only grow downward as it gets longer — never upward.

If a flow naturally wants to expand upward (e.g. two branches with no clear primary), restructure it so the visual flow stays below the title. The floor can drop as far as needed; the ceiling never moves.

## Decision placement

Decisions are the visual pivot points. Reserve more vertical space around them than around regular shapes (~80 px above and below the diamond) so the fan-out is readable. All extra vertical space goes **below** the diamond, never above — see "Title is the ceiling" above.

## Page-level positioning

When rendering a new diagram on a FigJam page that already has content, place the new diagram **clear of existing nodes**:

1. Scan `figma.currentPage.children` for the rightmost X coordinate of existing content.
2. Position the title at `rightmostX + 400` to leave a wide visual buffer.
3. Vertically align the title's baseline to the topmost existing node, or to `y = 0` if the page is empty.

Never place the new diagram at `(0, 0)` — FigJam appends new nodes there by default, and they will overlap.

## Title and legend positioning

Positions are derived **bottom-up from the title**, not from a fixed `originY`. This is the only way to guarantee the legend never overlaps the tallest shape in row 1, since row-1 height isn't known until after `fitShape()` runs on every row-1 shape.

The derivation order is:

1. Place the **title** at the top, at the diagram's chosen `originX` / `originY`. The `originY` is the title's **top edge**.
2. `legendY = title.y + title.height + 48` — legend's top edge.
3. `legendBottomY = legendY + legendHeight` — `legendHeight` is the legend group's bounding-box height (≈ 24 px for a single-line legend, but measure the actual group rather than assuming).
4. Run `fitShape()` on every shape that will go in row 1. Compute `row1MaxHeight = max(shape.height for shape in row1)`.
5. `row1TopY = legendBottomY + 80` — top edge of row 1.
6. `row1CenterY = row1TopY + row1MaxHeight / 2`.
7. Every shape in row 1 is placed at `shape.y = row1CenterY - shape.height / 2` (center-aligned to `row1CenterY`).
8. For subsequent rows (when snaking), `rowNCenterY = prevRowCenterY + (prevRowMaxHeight + nextRowMaxHeight) / 2 + 160`.

The key change versus a fixed offset: **`row1CenterY` depends on `row1MaxHeight`**. A row of tall 4-line shapes lands further below the legend than a row of compact 1-line shapes, but the legend-to-top-of-tallest-shape gap stays a constant 80 px in both cases.

All `x` positions still anchor to the same `originX` (the title's left edge).

## Alignment

**All shapes in a row are vertically center-aligned to each other.** A row has a single horizontal axis line; every shape's vertical center sits on that axis, regardless of its individual height. So a 184-px-tall diamond and a 112-px-tall square in the same row end up offset in `y` (the diamond's `y` is lower) but their *centers* are aligned.

Compute each shape's `y` as:

```
shape.y = rowCenterY - (shape.height / 2)
```

where `rowCenterY` is constant for the whole row. This keeps connectors clean (`magnet: 'RIGHT'` → `magnet: 'LEFT'` exits the right-middle and enters the left-middle) and makes the row feel composed rather than top-stuck.

Outcome ellipses that drop below the row (e.g. the "Positive" branch below a binary decision diamond) sit on their own sub-axis below the main `rowCenterY` — typically `rowCenterY + diamond.height/2 + 52`. Their centers align to each other on that sub-axis, same rule.

### Connectors must be colinear, or they kink

An ELBOWED connector inserts a right-angle jog whenever its two attach points aren't on the same axis. A tiny vertical offset between two shapes that are *meant* to be in a straight line produces an ugly little S-bend corner. **The fix is alignment, not line type** — never paper over a kink by switching to STRAIGHT; that just hides a real misalignment that will resurface elsewhere.

The invariant: **any two shapes joined by a straight horizontal (or straight vertical) connector must have exactly equal center coordinates on the perpendicular axis.**

- **Horizontal hop** (shape → next shape in the same row): both shapes must have the same vertical center. The per-row `rowCenterY` rule already guarantees this — so attach with explicit `magnet: 'RIGHT'` → `magnet: 'LEFT'`, which lands both endpoints at exactly `rowCenterY`. The connector is then perfectly horizontal and cannot kink.
- **Vertical drop** (decision → outcome directly below it, or any straight downward edge): both shapes must have the same horizontal center (`shape.x + shape.width/2` equal for both). Place the lower shape so its center-x matches the upper shape's center-x, then attach `magnet: 'BOTTOM'` → `magnet: 'TOP'`.

**Do not use `magnet: 'AUTO'` for straight runs.** AUTO lets FigJam pick attach points, and on shapes of differing size it can choose points that aren't colinear — producing the exact kink we're trying to avoid. Reserve AUTO (or explicit side magnets with ELBOWED) only for genuine direction changes where a corner is intended (fan-out branches, snake-wrap returns).

When you place a shape that will be connected by a straight edge, **derive its perpendicular-axis position from the shape it connects to**, not from an independent layout pass. A downstream shape's center-x should be copied from its predecessor when the edge between them is a vertical drop.
