# Typography

Three text roles: **title**, **legend**, **shape labels**. Each has a specific font and size.

## Title

The diagram's name plus a timestamp, e.g. `Acme Analytics Signup Flow (06.24.2026)`.

The name itself should be **descriptive** — combining the flow type with the product/path context. Not a raw URL, not a generic placeholder. See the title synthesis rules in [SKILL.md](../SKILL.md#the-cartographers-loop) for how to derive it from the user's target.

- **Font (FigJam)**: **`Merriweather`** — the typeface behind FigJam's "Bookish" style preset. Load `Merriweather Bold` for the flow name and `Merriweather Regular` for the timestamp. **No fallback in the FigJam path** — Merriweather ships with FigJam as the "Bookish" default, so assume availability. If it ever fails to load, stop and tell the user rather than degrading silently.
- **Font (HTML)**: Merriweather first, then a stack of genuinely-universal system serifs that don't depend on Google Fonts being reachable. Use this CSS stack: `'Merriweather', Georgia, 'Times New Roman', serif`. Georgia is the closest visual match to Merriweather among system fonts (preinstalled on macOS, Windows, ChromeOS, most Linux). Other Google Fonts (Lora, Source Serif Pro, etc.) are *not* useful fallbacks because they have the same delivery channel as Merriweather — if one fails, they all fail.
- **Size**: 64 px (display)
- **Color**: `#1E1E1E` (FigJam dark)
- **Weight — mixed run**:
  - The flow name itself is **Bold**.
  - The timestamp `(MM.DD.YYYY)` is **Regular** (not bold). Render the title as a single text node with a styled range — bold for the name, regular weight for the date suffix.
- **Timestamp**: always present, format `(MM.DD.YYYY)`, today's date in the user's locale. It reflects when the diagram was generated, not the date of the underlying source.

The title is the most visually prominent element on the canvas — it's how a viewer scanning a wall of FigJam content knows this diagram exists and what it depicts.

## Legend

A compact row beneath the title that maps colors to entities:

```
Entity:  ● Human    ● AI Model    ● Deterministic System
```

- **Font**: Inter, mixed weight on a single styled text node:
  - The leading label **`Entity:`** is **Bold**.
  - The entity names that follow (`Human`, `AI Model`, `Deterministic System`) are **Medium**.
- **Size**: 16 px (uniform across both weights)
- **Color**: `#1E1E1E` throughout
- **Dot**: 16 × 16 ELLIPSE shape, filled with the entity color and no stroke
- **Spacing**: 12 px between dot and label; 28 px between legend items
- **Order**: Human → AI Model → Deterministic System. Omit any entity not used in the flow.

Render the legend's text as **one mixed-weight text node** with `setRangeFontName` applied to the `Entity:` substring (Bold) and the entity-name substrings (Medium). Don't split the text into separate nodes — the spacing won't line up cleanly.

**Group the legend.** After creating the entity dots and the text node, wrap them all in a single FigJam **Group** (`figma.group([...legendNodes], figma.currentPage)`). The group should be named `Legend`. This keeps the legend movable as one unit and prevents accidental disassembly when the user pans / rearranges the diagram.

**Only the entity legend is rendered.** Do not add an outcome legend — positive/negative outcomes are self-explanatory from the green/red ellipses and their labels. Adding a legend for them is redundant noise.

## Shape labels

Text inside SHAPE_WITH_TEXT nodes (every shape on the canvas).

- **Font**: Inter Medium (this is FigJam's ShapeWithText default — never hardcode something else)
- **Size**: FigJam default (typically 16 px; do not override)
- **Color**: matches the entity preset's `text` color (`#1E1E1E` for all current palette entries)
- **Content**: All text the reader should see goes into `shape.text.characters` — do not split into a short label and a separate description field. The shape is auto-sized to fit the full string via the `fitShape` measurer pattern.

### Writing good labels

- **Verb-first for action shapes** (rounded rects, AI/Human squares): "Generate report," "Submit form," "Review proposal."
- **Noun for state shapes** (system squares, pages): "Settings page," "Onboarding banner," "Empty state."
- **Question for decisions**: "Subscription active?", "Credit balance > 0?"
- **Single word for outcomes**: "Positive," "Negative," or a concrete outcome name like "Approved" / "Rejected."

Keep labels short — two short lines is the target. If you need more than three lines, the step probably wants to be split or summarized.

## Connector labels

By default, connectors carry **no text**. The shapes they connect convey the meaning (a green ellipse already says "yes / positive").

When the user explicitly enables connector labels (see [render-figjam](render-figjam.md)), labels follow these rules:
- **Font**: Inter Medium
- **Size**: 14 px (slightly smaller than shape labels)
- **Color**: `#1E1E1E`
- **Content**: the trigger or condition, lowercased — "yes" / "no" / "payment added" / "key created"

Set `connector.text.fontName` *before* `connector.text.characters`. A new connector's `text.fontName` is invalid by default — see the FigJam plugin API gotchas.
