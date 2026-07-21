# Shape semantics

Shape answers "what kind of step is this?" Color (from [palette](palette.md)) answers "who owns it?" The two compose: a violet square is an AI action; a blue square is a human action; a gray square is a system state.

## Base shapes

These are the canonical seven. Use them first; only reach for additions when none of them fit.

| Shape | Meaning | Typical entity color | Notes |
|---|---|---|---|
| `CHEVRON` | **Flow starting point** — where this flow begins. | Human (light blue) | Always the leftmost / topmost node. One per diagram (or one per swim lane if you split). |
| `SQUARE` | **State or action** — a screen the user is on, a system state, an action taken. Generic workhorse. | Any entity | Page ≡ state for our purposes. A user-visible screen and an internal system state share the same shape. |
| `ROUNDED_RECTANGLE` | **Human action / reaction** — emphasizes a deliberate human step (button click, form submit, decision to leave). | Human (light blue) | Use this when you want to distinguish a *human action* from a generic state. If everything in the flow is human-driven, plain SQUARE is fine. |
| `DIAMOND` | **Decision point** — a branch in the flow. | Deterministic system (gray) if the system evaluates; Human (light blue) if the user evaluates. | **Must have ≥ 2 outgoing connectors.** A diamond with only one path is not a decision — it's a regular step. If you've drawn one with a single outgoing edge, either add the missing branch (it should have been there) or downgrade the shape to a SQUARE / ROUNDED_RECTANGLE. |
| `ELLIPSE` (small, ~102×102) | **Outcome of a decision** — Positive (green) or Negative (red). | Outcome colors only — never entity colors. | Place immediately after a DIAMOND. The valence is the meaning. |
| `SQUARE` (dashed stroke) | **Note / annotation** — an aside, a clarification, an external pointer that isn't part of the flow itself. | Transparent fill, gray dashed stroke. | Off the flow path; attach with a connector if it pertains to a specific step. |

## The composition rule

Whenever you place a shape, decide **shape first, then color**.

1. *What is this step?* → picks the shape.
2. *Who owns or executes it?* → picks the color (or outcome valence, for ellipses).

If a step doesn't cleanly fit into one of the base shapes, prefer SQUARE with a clear label over inventing a new shape.

## Candidate additions

The following are *proposed*, not canonical. They cover gaps the base set doesn't reach. Adopt them only when the gap actually appears in a flow — keeping the shape vocabulary small is more valuable than precision.

| Shape | Proposed meaning | When to use |
|---|---|---|
| `ENG_DATABASE` (cylinder) | Datastore — durable storage that the flow reads from or writes to. | A step persists user data, fetches from a DB, or hits a cache. Useful for "state changes" with persistence implications. |
| `PARALLELOGRAM_RIGHT` | Input / output — data crosses a system boundary (form submit returning a result, API call returning JSON). | Use sparingly. Most "API calls" are already covered by a violet SQUARE (AI action) or gray SQUARE (system action). Reach for parallelogram only when the input/output framing is the *point*. |
| `DOCUMENT_SINGLE` (wavy bottom) | Document or report generated. | A flow that emits an artifact (export, receipt, PDF, summary report). |
| `SPEECH_BUBBLE` | Notification or message out. | The system sends an email, push notification, Slack message, or similar message *to* the user. Distinct from a screen the user views. |
| `ENG_QUEUE` (horizontal cylinder) | Async work / job queue. | A step kicks off background work that completes later. |
| `TRAPEZOID` | Manual step or off-system operation. | A step the user does *outside* the product (calls support, signs a paper form, talks to a teammate). |
| `PENTAGON` (off-page connector) | "Continues elsewhere." | The flow continues on another diagram. Pair with a label that names the destination diagram. |

When you add one of these to a flow, **also add a legend entry** for it so readers know what the new shape means.

## What we deliberately don't have

- **Separate shapes for "page" vs. "state."** They're the same in this system; the label tells you which framing applies.
- **Loop-back symbols.** Use a connector that arcs back to an earlier node. If the arc is hard to follow, split into two diagrams and use an off-page connector.
- **Swimlanes.** Vertical lanes by entity are overkill given color already encodes entity. Reach for a swimlane only if a flow has so many entity transitions that the connectors become spaghetti.

## Historical reference

The base vocabulary draws from ISO 5807 (1985) flowchart symbols, modernized for user-flow rather than computational-flow context. Specifically:
- ISO `Process` (rectangle) → SQUARE
- ISO `Decision` (rhombus) → DIAMOND
- ISO `Terminator` (oval/stadium) → CHEVRON for start, ELLIPSE for outcomes
- ISO `Document` (wavy bottom) → DOCUMENT_SINGLE (candidate)
- ISO `Manual operation` (trapezoid) → TRAPEZOID (candidate)
- ISO `Database` (cylinder) → ENG_DATABASE (candidate)

The departure from ISO is using **shape to mean step-kind** and **color to mean owner-entity** — ISO is monochrome. This is what makes the diagrams readable for AI-era flows where the same action could be performed by a human, an AI, or a deterministic system.
