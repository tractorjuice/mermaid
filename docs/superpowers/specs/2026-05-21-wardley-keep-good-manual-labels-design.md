# Wardley Autoplace — Keep Good Manual Labels

**Date:** 2026-05-21
**Status:** Design approved, pending implementation plan
**Diagram:** `wardley-beta` (`packages/mermaid/src/diagrams/wardley/`)
**Builds on:** [2026-05-20 Wardley autoplace labels](./2026-05-20-wardley-autoplace-labels-design.md)

## Problem

The `autoPlaceLabels` feature currently **ignores manual `label [x, y]` offsets
entirely** — when the flag is on, every label is recomputed from scratch. Tested
against real-world Wardley maps, this is the feature's biggest weakness:
carefully authored maps have per-label tuning (`label [-101, 4]`, etc.) that
auto-placement discards, often producing a worse result than no autoplacement at
all.

This change makes autoplacement **respect a manual label position when that
position already works** — i.e. when the authored label does not overlap
anything. Manual labels that *do* collide are still re-placed, but biased to stay
near where the author put them.

## Goals

- A manual `label [x, y]` whose position is collision-free is kept exactly.
- A kept manual label acts as a fixed obstacle, so auto-placed labels route
  around it.
- A manual label that collides is re-placed, biased toward the author's
  intended position (minimum movement to clear the collision).
- Deterministic, order-independent classification.
- Zero behaviour change when `autoPlaceLabels` is off.

## Non-goals

- Link labels and annotations — they have no manual-offset concept, so they are
  unaffected and remain fully auto-placed.
- Any change to the greedy algorithm, refinement pass, or leader-line logic
  beyond operating on a subset of labels.
- A config option to choose between keep/override behaviour — keep-if-good is
  simply the new behaviour of the flag.

## Decisions

| Decision | Choice |
|----------|--------|
| When is a manual label kept? | When its authored position is collision-free |
| Kept manual label role | Fixed obstacle for auto-placed labels |
| Collision check scope | Static obstacles + other manual labels' authored rects |
| Two manual labels overlapping each other | Neither is kept; both re-placed |
| A colliding manual label | Re-placed, biased toward the author's position |
| Own node marker | Excluded from that label's own collision check |

## Architecture — Approach A

The classification is geometric, so it lives in the pure, unit-tested
`wardleyLabelPlacement.ts` module alongside the existing geometry. The renderer
only supplies the authored position.

### Data model

`LabelBox` gains one optional field:

```ts
manualRect?: Rect; // the author's chosen label position
```

It is set only for component/anchor labels that have an explicit `label [x, y]`.
Absent for untuned labels, link labels, and annotations.

### `autoPlaceLabels` flow

1. **Partition** input labels into *manual* (has `manualRect`) and *untuned*.
2. **Classify** each manual label. It is **kept** iff its `manualRect` collides
   with nothing (see Collision check). The check is symmetric — if manual A and
   manual B overlap, neither is kept — so the partition is order-independent.
3. **Kept** manual labels are emitted directly as a `PlacedLabel` with
   `rect = manualRect` and `needsLeader = false`. Their rects are added to the
   obstacle set.
4. **Pool** = untuned labels + not-kept manual labels. The pool is placed by the
   existing greedy algorithm + refinement pass, with obstacles = static
   obstacles + kept manual rects. A not-kept manual label carries its
   `manualRect` as a placement bias (see Bias).
5. Output preserves original input order.

The greedy placer, refinement pass, and leader-line logic are unchanged; they
operate on the pool instead of all labels.

## Collision check

A manual label's `manualRect` **collides** if any of:

- It overlaps a node marker of any **other** node (`circleRectOverlap`).
- It overlaps a **rect obstacle** such as a pipeline box (`rectsOverlapArea > 0`).
- It overlaps **another manual label's** `manualRect` (`rectsOverlapArea > 0`).
- Any part lies **outside the chart boundary** (`areaOutsideBounds > 0`).

**Link lines are not a disqualifier.** A manual label that merely crosses a
link-line segment is still kept. Link lines are thin; an author who placed a
label across one accepted that, and on dense maps treating every clipped link
as a collision re-places labels that looked fine. Link segments remain a *soft*
penalty for auto-placed (untuned) labels via `scoreCandidate`.

**Exception — own node marker.** A manual label is *not* judged as colliding
with its own node's marker. The default offset (~8px from a 6px-radius node)
puts a normal label snugly against its own marker; counting that as a collision
would reject almost every tightly authored label. The own marker is still an
obstacle for *other* labels.

**Pipeline boxes.** The rounded rectangle drawn around a pipeline's child
components is fed into placement as a `rect` obstacle, so labels — auto-placed
or manual — are kept off the box.

The check evaluates every manual label against the same fixed set (static
obstacles + all other manual rects), so the kept/not-kept partition does not
depend on evaluation order.

## Bias — re-placing a colliding manual label

A not-kept manual label goes through the greedy placer with its preference
retargeted toward the authored spot:

- **Candidate set:** the usual 8-compass-direction slots around the node, plus
  the `manualRect` itself as one extra candidate.
- **Scoring:** `scoreCandidate` gains an optional `preferredCenter` parameter.
  Today its two soft (low-weight) terms are distance-from-anchor and
  deviation-from-NE-direction. When `preferredCenter` is supplied (the
  `manualRect` center), those two are replaced by a single soft penalty
  proportional to distance from `preferredCenter`. The hard terms — overlap with
  labels, markers, boundary, link lines — keep their existing high weights,
  unchanged.

Effect: the placer still hard-avoids every collision, but among acceptable slots
picks the one closest to the authored position — minimum movement to clear the
collision. Untuned labels (no `preferredCenter`) score exactly as before.
Leader lines apply normally: a re-placed manual label still far from its node
gets a leader.

## Renderer integration

`buildPlacement` in `wardleyRenderer.ts` changes in one spot: when building the
`LabelBox` for a node that has a manual offset (`labelOffsetX`/`labelOffsetY`
defined — the parser sets them as a pair from `label [x, y]`), it also computes
`manualRect` — the bounding box of the label as the legacy offset path would
render it (author's `pos + offset`, sized by `estimateLabelBox`, accounting for
the component-vs-anchor text-anchor and baseline). The `manualRect` is attached
to the `LabelBox`.

Nothing else in the renderer changes. The apply step is already uniform — every
label is positioned from its returned `PlacedLabel.rect`. A kept manual label
returns with `rect === manualRect`, so the existing rect-based positioning
reproduces the author's location with no special-casing. The flag-off path is
untouched.

## Testing

- **Unit tests** (`wardleyLabelPlacement.spec.ts`):
  - A collision-free manual label is kept — output rect equals `manualRect`,
    `needsLeader` false.
  - A manual label overlapping another node's marker is re-placed and ends up
    collision-free.
  - Two mutually-overlapping manual labels are both re-placed.
  - A kept manual label acts as an obstacle — an untuned label routes around it.
  - A re-placed manual label lands closer to its `manualRect` than a default NE
    placement would.
  - A manual label overlapping *only its own* node marker is still kept.
  - Determinism — identical input yields identical output.
- **Cypress visual test**: a map mixing good manual labels, colliding manual
  labels, and untuned labels, rendered with `autoPlaceLabels: true`.
- Confirm the flag-off path and existing snapshots are unchanged.

## Files touched

| File | Change |
|------|--------|
| `wardleyLabelPlacement.ts` | `LabelBox.manualRect`; partition/classify in `autoPlaceLabels`; `preferredCenter` in `scoreCandidate`; manual-rect candidate |
| `wardleyLabelPlacement.spec.ts` | Unit tests for keep/discard/bias |
| `wardleyRenderer.ts` | `buildPlacement` computes `manualRect` for manually-offset nodes |
| `cypress/integration/rendering/wardley/wardley.spec.js` | Mixed manual/untuned visual test |
