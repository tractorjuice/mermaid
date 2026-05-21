# Wardley Diagram — Automatic Label Placement

**Date:** 2026-05-20
**Status:** Design approved, pending implementation plan
**Diagram:** `wardley-beta` (`packages/mermaid/src/diagrams/wardley/`)

## Problem

Wardley maps place every label with a fixed static offset (≈8px up/right of the
node). On any non-trivial map, labels overlap each other, cover node markers, and
spill outside the chart. The only fix today is manually specifying `label [x, y]`
offsets in the DSL for each affected label — tedious and brittle.

This feature adds opt-in automatic label placement that repositions labels to
avoid collisions.

## Goals

- Automatically place **all** label types: component, anchor, link, and
  annotation labels.
- Avoid overlapping: other labels, node markers, the chart boundary, and link
  lines.
- Deterministic output — mermaid relies on visual snapshot tests.
- Zero impact on existing maps unless the feature is explicitly enabled.

## Non-goals

- Generic, cross-diagram label placement. This is wardley-local; extract later
  if a second caller appears (YAGNI).
- Exposing tuning parameters (weights, distances, thresholds) as config.
- Text wrapping / multi-line labels.

## Decisions

| Decision | Choice |
|----------|--------|
| Scope | All label types — component, anchor, link, annotation |
| Manual `label [x,y]` offsets | Fully ignored when autoplacement is enabled |
| Activation | Opt-in via config flag `wardley.autoPlaceLabels`, default `false` |
| Obstacles avoided | Other labels, node markers, chart boundary, link lines |
| Algorithm | Greedy candidate-slot placement (deterministic) |
| Leader lines | Drawn only when a label is moved past a distance threshold |
| Text width | Estimated via character-width model (validated against `getBBox`) |

## Architecture

### New file: `wardleyLabelPlacement.ts`

A pure, DOM-free, d3-free module — the unit-tested geometric core.

```ts
autoPlaceLabels(
  labels: LabelBox[],
  obstacles: Obstacle[],
  bounds: Rect,
  config: PlacementConfig
): PlacedLabel[]
```

- `LabelBox` — anchor point `{x, y}`, estimated `width`/`height`, `kind`
  (`component` | `anchor` | `link` | `annotation`), and a `priority` derived
  from source-declaration order.
- `Obstacle` — node markers (circles), link line segments, and the chart
  boundary rect.
- `PlacedLabel` — final `{x, y}` plus a `needsLeader` flag.

A pure character-width estimator (also in this module or a sibling) provides
`width`/`height`, so the entire feature is deterministically unit-testable
without a browser.

### Modified file: `wardleyRenderer.ts`

- When `autoPlaceLabels` is on: build `LabelBox`/`Obstacle` arrays, call
  `autoPlaceLabels`, apply the returned `x`/`y` to each text element, and draw
  leader lines where `needsLeader` is set.
- When off: the existing static-offset code path runs unchanged.

### Width estimation — validation step

Before relying on estimation, the implementation must compare estimator output
against real `getBBox()` measurements across the existing wardley example maps.
If estimation error is small enough that no visible overlaps/gaps appear, keep
it. Otherwise, promote a `getBBox`-based measurement path as a fallback. This
decision is made on recorded evidence, not assumption.

## Algorithm

### Candidate slots

For each label, generate candidate positions around its anchor:

- **Component & anchor labels:** 8 compass directions × 2 distances (~16
  candidates), biased toward up-right (the current default) so sparse maps look
  unchanged.
- **Link labels:** candidates along the perpendicular on both sides of the link
  midpoint.
- **Annotation boxes:** a small grid of offsets around the declared position.

### Scoring

Each candidate's penalty is a weighted sum of:

- overlap area with already-placed labels — **high**
- overlap with node markers — **high**
- area outside the chart boundary — **very high** (effectively forbidden)
- intersections with link line segments — **medium**
- distance from anchor / leader length — **low** (prefer close)
- deviation from the preferred direction — **low** (tie-breaker)

Weights are internal constants.

### Placement order (greedy, deterministic)

1. Sort labels most-constrained-first — fewest zero-penalty candidates —
   breaking ties by node-declaration order.
2. Place each label at its lowest-penalty slot.
3. Once placed, a label becomes an obstacle for all subsequent labels.

### Refinement pass

After the first pass, take the `N` highest-penalty labels and re-place them
against the now-complete layout. One pass, fixed `N` — bounded and deterministic.

### Leader lines

If a placed label's distance from its anchor exceeds a threshold, set
`needsLeader`. The renderer draws a thin line from the node to the label edge.

## Configuration

```ts
wardley: {
  autoPlaceLabels: boolean // default: false
}
```

- Added to `schemas/config.schema.yaml` with a doc description.
- Read by the renderer's `getConfigValues()`.
- `false` → existing behavior, zero snapshot churn.
- `true` → all labels auto-placed; manual `label [x,y]` offsets ignored.

## Testing

- **Unit tests** (`wardleyLabelPlacement.spec.ts`) — pure core with synthetic
  boxes: two labels that must separate; a label pushed off-boundary;
  most-constrained ordering; leader-line threshold; determinism (same input →
  same output); empty/degenerate input.
- **Width-estimator validation** — estimator vs `getBBox()` on existing example
  maps, recorded in implementation notes.
- **Cypress visual tests** (`cypress/integration/rendering/wardley/wardley.spec.js`)
  — a new dense example map rendered with `autoPlaceLabels: true`; plus
  confirmation that an existing map with `autoPlaceLabels` unset is
  byte-identical.

### Edge cases

- Map with zero or one label.
- All labels colliding at a single point — algorithm degrades gracefully, never
  throws.
- Labels larger than the chart area.

## Files touched

| File | Change |
|------|--------|
| `wardleyLabelPlacement.ts` | New — pure placement core + width estimator |
| `wardleyLabelPlacement.spec.ts` | New — unit tests |
| `wardleyRenderer.ts` | Integrate placement, draw leader lines, gate on flag |
| `schemas/config.schema.yaml` | Add `wardley.autoPlaceLabels` |
| `cypress/integration/rendering/wardley/wardley.spec.js` | New dense example test |
