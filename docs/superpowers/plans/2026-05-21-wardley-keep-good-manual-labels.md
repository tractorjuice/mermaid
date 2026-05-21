# Wardley Keep-Good-Manual-Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the wardley-beta `autoPlaceLabels` feature keep a manual `label [x, y]` position when that position is collision-free, instead of always overriding it.

**Architecture:** `LabelBox` gains an optional `manualRect` (the author's chosen position). Inside the pure `autoPlaceLabels` function, manual labels are partitioned out: a collision-free one is emitted unchanged and added to the obstacle set; a colliding one joins the placement pool but is biased — via a new `preferredCenter` term in `scoreCandidate` and a manual-position candidate slot — to move only as far as needed. The renderer computes `manualRect` for manually-offset nodes; nothing else in the renderer changes.

**Tech Stack:** TypeScript, D3 (SVG rendering), Vitest (unit tests), Cypress (visual regression).

---

## File Structure

| File | Responsibility |
|------|----------------|
| `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts` | MODIFY — `LabelBox.manualRect`; `preferredCenter` in `scoreCandidate`; manual-position candidate in `generateCandidates`; classification helper + keep/pool logic in `autoPlaceLabels`. |
| `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts` | MODIFY — unit tests for the new behaviour. |
| `packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts` | MODIFY — `buildPlacement` computes `manualRect` for manually-offset nodes. |
| `cypress/integration/rendering/wardley/wardley.spec.js` | MODIFY — visual test mixing good manual, colliding manual, and untuned labels. |

**Test runner:** `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts` from `/workspaces/mermaid`.

**Current state (for reference):** `wardleyLabelPlacement.ts` already exports `LabelBox`, `Candidate`, `Obstacle`, `PlacementConfig`, `PlacedLabel`, `scoreCandidate`, `generateCandidates`, `autoPlaceLabels`, and the geometry helpers `rectsOverlapArea`, `circleRectOverlap`, `segmentIntersectsRect`, `areaOutsideBounds`. `scoreCandidate`'s current signature is `(candidate, obstacles, bounds, anchor, placedRects = [])`.

---

## Task 1: Add `manualRect` to `LabelBox` and `preferredCenter` to `scoreCandidate`

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `wardleyLabelPlacement.spec.ts` (the `scoreCandidate` and `Candidate`/`Rect` imports already exist at the top of the file — reuse them):

```ts
describe('scoreCandidate preferredCenter bias', () => {
  const bounds = { x: 0, y: 0, width: 400, height: 400 };
  const near = {
    rect: { x: 100, y: 100, width: 40, height: 12 },
    direction: { x: 1, y: 0 },
    distance: 50,
  };
  const far = {
    rect: { x: 300, y: 300, width: 40, height: 12 },
    direction: { x: 1, y: 0 },
    distance: 50,
  };

  it('prefers the candidate closer to preferredCenter when one is given', () => {
    const preferred = { x: 120, y: 106 }; // center of `near`'s rect
    const nearScore = scoreCandidate(near, [], bounds, { x: 0, y: 0 }, [], preferred);
    const farScore = scoreCandidate(far, [], bounds, { x: 0, y: 0 }, [], preferred);
    expect(nearScore).toBeLessThan(farScore);
  });

  it('ignores direction/anchor-distance terms when preferredCenter is given', () => {
    // Two candidates equidistant from preferredCenter but different compass
    // directions must score equally — the NE-direction bias is suppressed.
    const preferred = { x: 200, y: 200 };
    const east = {
      rect: { x: 240, y: 194, width: 40, height: 12 },
      direction: { x: 1, y: 0 },
      distance: 60,
    };
    const west = {
      rect: { x: 120, y: 194, width: 40, height: 12 },
      direction: { x: -1, y: 0 },
      distance: 60,
    };
    expect(scoreCandidate(east, [], bounds, { x: 0, y: 0 }, [], preferred)).toBeCloseTo(
      scoreCandidate(west, [], bounds, { x: 0, y: 0 }, [], preferred)
    );
  });

  it('still applies hard obstacle penalties with preferredCenter set', () => {
    const preferred = { x: 120, y: 106 };
    const blocked = scoreCandidate(
      near,
      [{ type: 'circle', x: 120, y: 106, radius: 8 }],
      bounds,
      { x: 0, y: 0 },
      [],
      preferred
    );
    const clear = scoreCandidate(near, [], bounds, { x: 0, y: 0 }, [], preferred);
    expect(blocked).toBeGreaterThan(clear + 100);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts -t "preferredCenter"`
Expected: FAIL — `scoreCandidate` does not accept a 6th argument / the bias is not applied.

- [ ] **Step 3: Add `manualRect` to `LabelBox`**

In `wardleyLabelPlacement.ts`, add the field to the `LabelBox` interface, after `preferredOffset`:

```ts
export interface LabelBox {
  /** Stable identifier — used to key results back to render elements. */
  id: string;
  /** The point the label is attached to (node center / link midpoint / declared annotation position). */
  anchor: Point;
  width: number;
  height: number;
  kind: LabelKind;
  /** Source-declaration order; lower is placed-priority earlier on ties. */
  priority: number;
  /** For link labels: unit vector hint for the perpendicular side preference. */
  preferredOffset?: Point;
  /** Author's chosen label position (`label [x, y]`). Component/anchor labels only. */
  manualRect?: Rect;
}
```

- [ ] **Step 4: Add the `preferredCenter` weight constant**

In `wardleyLabelPlacement.ts`, next to the other weight constants (after `WEIGHT_DIRECTION = 6;`):

```ts
const WEIGHT_DISTANCE = 0.05;
const WEIGHT_DIRECTION = 6;
// Soft pull toward an author-specified position; replaces the distance +
// direction terms when a candidate is scored against a manual label.
const WEIGHT_PREFERRED = 0.05;
```

- [ ] **Step 5: Add the `preferredCenter` parameter to `scoreCandidate`**

Replace the current `scoreCandidate` with this version. The hard terms are unchanged; only the trailing soft terms branch on `preferredCenter`:

```ts
/**
 * Score a candidate position. Lower is better. A score near 0 is an
 * unobstructed placement close to the anchor in the preferred direction.
 * When `preferredCenter` is supplied, the soft distance/direction terms are
 * replaced by a single pull toward that point (used to bias a re-placed
 * manual label back toward the author's intended position).
 */
export const scoreCandidate = (
  candidate: Candidate,
  obstacles: Obstacle[],
  bounds: Rect,
  anchor: Point,
  placedRects: Rect[] = [],
  preferredCenter?: Point
): number => {
  let penalty = 0;
  const { rect } = candidate;

  for (const placed of placedRects) {
    penalty += rectsOverlapArea(rect, placed) * WEIGHT_LABEL_OVERLAP;
  }

  for (const obstacle of obstacles) {
    if (obstacle.type === 'circle') {
      if (circleRectOverlap(obstacle, rect)) {
        penalty += WEIGHT_MARKER_OVERLAP;
      }
    } else if (segmentIntersectsRect(obstacle, rect)) {
      penalty += WEIGHT_LINK_CROSS;
    }
  }

  penalty += areaOutsideBounds(rect, bounds) * WEIGHT_OUT_OF_BOUNDS;

  if (preferredCenter) {
    // Soft pull toward the author's intended position.
    const cx = rect.x + rect.width / 2;
    const cy = rect.y + rect.height / 2;
    penalty += Math.hypot(cx - preferredCenter.x, cy - preferredCenter.y) * WEIGHT_PREFERRED;
  } else {
    penalty += candidate.distance * WEIGHT_DISTANCE;
    // Direction deviation: 0 when aligned with preferred, up to 2 when opposite.
    const dot =
      candidate.direction.x * PREFERRED_DIRECTION.x + candidate.direction.y * PREFERRED_DIRECTION.y;
    penalty += (1 - dot) * WEIGHT_DIRECTION;
  }

  return penalty;
};
```

Note: `anchor` remains an unused parameter (it was already unused — kept for signature stability and future use). Do not rename it.

- [ ] **Step 6: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — the new `preferredCenter` tests pass and all pre-existing tests still pass.

- [ ] **Step 7: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): add preferredCenter bias to candidate scoring"
```

---

## Task 2: Include the manual position as a candidate slot

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `wardleyLabelPlacement.spec.ts`:

```ts
describe('generateCandidates with manualRect', () => {
  it('appends the manual rect as an extra candidate', () => {
    const label: LabelBox = {
      id: 'm',
      anchor: { x: 100, y: 100 },
      width: 40,
      height: 12,
      kind: 'component',
      priority: 0,
      manualRect: { x: 200, y: 50, width: 40, height: 12 },
    };
    const withManual = generateCandidates(label, [12, 24]);
    // 8 compass directions x 2 distances = 16, plus the manual rect = 17.
    expect(withManual).toHaveLength(17);
    const manualCandidate = withManual.find(
      (c) => c.rect.x === 200 && c.rect.y === 50
    );
    expect(manualCandidate).toBeDefined();
  });

  it('the manual candidate carries anchor-relative direction and distance', () => {
    const label: LabelBox = {
      id: 'm',
      anchor: { x: 100, y: 100 },
      width: 40,
      height: 12,
      kind: 'component',
      priority: 0,
      // rect center is (100, 100) + (50, 0) -> straight east, distance 50.
      manualRect: { x: 80, y: 94, width: 40, height: 12 },
    };
    const manualCandidate = generateCandidates(label, [12])
      .find((c) => c.rect.x === 80 && c.rect.y === 94);
    expect(manualCandidate).toBeDefined();
    expect(manualCandidate.distance).toBeCloseTo(0);
  });

  it('does not append a manual candidate when manualRect is absent', () => {
    const label: LabelBox = {
      id: 'u',
      anchor: { x: 100, y: 100 },
      width: 40,
      height: 12,
      kind: 'component',
      priority: 0,
    };
    expect(generateCandidates(label, [12, 24])).toHaveLength(16);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts -t "generateCandidates with manualRect"`
Expected: FAIL — `generateCandidates` returns 16, not 17 (no manual candidate appended).

- [ ] **Step 3: Append the manual candidate in `generateCandidates`**

In `wardleyLabelPlacement.ts`, replace `generateCandidates` with this version (the compass/perpendicular loop is unchanged; a manual-rect candidate is appended at the end):

```ts
/**
 * Generate the set of candidate positions for a label.
 * Component / anchor / annotation labels use the 8 compass directions at each
 * supplied distance. Link labels use only the perpendicular axis (both sides).
 * When the label has a `manualRect`, the author's exact position is appended
 * as one extra candidate.
 */
export const generateCandidates = (label: LabelBox, distances: number[]): Candidate[] => {
  const candidates: Candidate[] = [];
  const directions =
    label.kind === 'link' ? perpendicularDirections(label.preferredOffset) : COMPASS;
  for (const distance of distances) {
    for (const direction of directions) {
      const center = {
        x: label.anchor.x + direction.x * distance,
        y: label.anchor.y + direction.y * distance,
      };
      candidates.push({
        rect: rectAround(center, label.width, label.height),
        direction,
        distance,
      });
    }
  }
  if (label.manualRect) {
    const m = label.manualRect;
    const cx = m.x + m.width / 2;
    const cy = m.y + m.height / 2;
    const dx = cx - label.anchor.x;
    const dy = cy - label.anchor.y;
    const distance = Math.hypot(dx, dy);
    const direction =
      distance < Number.EPSILON ? { x: 0, y: 0 } : { x: dx / distance, y: dy / distance };
    candidates.push({ rect: { ...m }, direction, distance });
  }
  return candidates;
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — the new tests pass and all pre-existing tests still pass.

- [ ] **Step 5: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): include manual label position as a candidate"
```

---

## Task 3: Classification helper — is a manual label collision-free?

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

This adds a pure helper that decides whether one manual label's authored position is collision-free. It is exported so it can be unit-tested directly.

- [ ] **Step 1: Write the failing test**

Append to `wardleyLabelPlacement.spec.ts` (add `isManualLabelKept` to the existing import from `./wardleyLabelPlacement.js`):

```ts
describe('isManualLabelKept', () => {
  const bounds = { x: 0, y: 0, width: 400, height: 400 };
  const makeManual = (manualRect, anchor = { x: 50, y: 50 }): LabelBox => ({
    id: 'm',
    anchor,
    width: manualRect.width,
    height: manualRect.height,
    kind: 'component',
    priority: 0,
    manualRect,
  });

  it('keeps a manual label that overlaps nothing', () => {
    const label = makeManual({ x: 100, y: 100, width: 40, height: 12 });
    expect(isManualLabelKept(label, [], bounds, [])).toBe(true);
  });

  it('rejects a manual label that overlaps another node marker', () => {
    const label = makeManual({ x: 100, y: 100, width: 40, height: 12 });
    const obstacles: Obstacle[] = [{ type: 'circle', x: 110, y: 106, radius: 8 }];
    expect(isManualLabelKept(label, obstacles, bounds, [])).toBe(false);
  });

  it('keeps a manual label that overlaps only its OWN node marker', () => {
    // The label's anchor is its own node centre; a marker there must be ignored.
    const label = makeManual({ x: 40, y: 40, width: 40, height: 12 }, { x: 50, y: 50 });
    const obstacles: Obstacle[] = [{ type: 'circle', x: 50, y: 50, radius: 8 }];
    expect(isManualLabelKept(label, obstacles, bounds, [])).toBe(true);
  });

  it('rejects a manual label that crosses a link segment', () => {
    const label = makeManual({ x: 100, y: 100, width: 40, height: 12 });
    const obstacles: Obstacle[] = [
      { type: 'segment', x1: 90, y1: 90, x2: 160, y2: 130 },
    ];
    expect(isManualLabelKept(label, obstacles, bounds, [])).toBe(false);
  });

  it('rejects a manual label that spills outside the chart bounds', () => {
    const label = makeManual({ x: -20, y: 100, width: 40, height: 12 });
    expect(isManualLabelKept(label, [], bounds, [])).toBe(false);
  });

  it('rejects a manual label that overlaps another manual label', () => {
    const label = makeManual({ x: 100, y: 100, width: 40, height: 12 });
    const otherManualRects: Rect[] = [{ x: 110, y: 104, width: 40, height: 12 }];
    expect(isManualLabelKept(label, [], bounds, otherManualRects)).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts -t "isManualLabelKept"`
Expected: FAIL — `isManualLabelKept` is not exported.

- [ ] **Step 3: Implement `isManualLabelKept`**

In `wardleyLabelPlacement.ts`, add this function after `scoreCandidate` (it uses the geometry helpers already defined at the top of the file):

```ts
/**
 * Decide whether a manual label's authored position (`manualRect`) is
 * collision-free and may be kept as-is. A manual label is kept unless its
 * rect overlaps another node's marker, crosses a link segment, spills outside
 * the chart bounds, or overlaps another manual label's rect.
 *
 * The label's OWN node marker — the circle centred on `label.anchor` — is
 * ignored: a normal label sits snugly against its own node, and counting that
 * as a collision would reject almost every tightly authored label.
 *
 * `otherManualRects` must exclude this label's own `manualRect`.
 */
export const isManualLabelKept = (
  label: LabelBox,
  obstacles: Obstacle[],
  bounds: Rect,
  otherManualRects: Rect[]
): boolean => {
  const rect = label.manualRect;
  if (!rect) {
    return false;
  }
  if (areaOutsideBounds(rect, bounds) > 0) {
    return false;
  }
  for (const obstacle of obstacles) {
    if (obstacle.type === 'circle') {
      // Skip the label's own node marker.
      if (obstacle.x === label.anchor.x && obstacle.y === label.anchor.y) {
        continue;
      }
      if (circleRectOverlap(obstacle, rect)) {
        return false;
      }
    } else if (segmentIntersectsRect(obstacle, rect)) {
      return false;
    }
  }
  for (const other of otherManualRects) {
    if (rectsOverlapArea(rect, other) > 0) {
      return false;
    }
  }
  return true;
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — the new `isManualLabelKept` tests pass and all pre-existing tests still pass.

- [ ] **Step 5: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): add manual-label collision classifier"
```

---

## Task 4: Wire keep/pool/bias into `autoPlaceLabels`

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `wardleyLabelPlacement.spec.ts`:

```ts
describe('autoPlaceLabels with manual labels', () => {
  const bounds = { x: 0, y: 0, width: 400, height: 400 };
  const config: PlacementConfig = {
    slotDistances: [12, 24, 40],
    leaderThreshold: 30,
    refinementCount: 2,
  };

  it('keeps a collision-free manual label exactly where the author put it', () => {
    const labels: LabelBox[] = [
      {
        id: 'm',
        anchor: { x: 100, y: 100 },
        width: 40,
        height: 12,
        kind: 'component',
        priority: 0,
        manualRect: { x: 200, y: 60, width: 40, height: 12 },
      },
    ];
    const [placed] = autoPlaceLabels(labels, [], bounds, config);
    expect(placed.rect).toEqual({ x: 200, y: 60, width: 40, height: 12 });
    expect(placed.needsLeader).toBe(false);
  });

  it('re-places a manual label whose authored position overlaps a marker', () => {
    const obstacles: Obstacle[] = [{ type: 'circle', x: 215, y: 66, radius: 10 }];
    const labels: LabelBox[] = [
      {
        id: 'm',
        anchor: { x: 100, y: 100 },
        width: 40,
        height: 12,
        kind: 'component',
        priority: 0,
        manualRect: { x: 200, y: 60, width: 40, height: 12 },
      },
    ];
    const [placed] = autoPlaceLabels(labels, obstacles, bounds, config);
    // It moved off the authored position...
    expect(placed.rect).not.toEqual({ x: 200, y: 60, width: 40, height: 12 });
    // ...and the result no longer overlaps the marker.
    expect(circleRectOverlap(obstacles[0], placed.rect)).toBe(false);
  });

  it('re-places both of two manual labels that overlap each other', () => {
    const labels: LabelBox[] = [
      {
        id: 'a',
        anchor: { x: 100, y: 100 },
        width: 40,
        height: 12,
        kind: 'component',
        priority: 0,
        manualRect: { x: 200, y: 60, width: 40, height: 12 },
      },
      {
        id: 'b',
        anchor: { x: 300, y: 300 },
        width: 40,
        height: 12,
        kind: 'component',
        priority: 1,
        manualRect: { x: 210, y: 64, width: 40, height: 12 },
      },
    ];
    const placed = autoPlaceLabels(labels, [], bounds, config);
    expect(rectsOverlapArea(placed[0].rect, placed[1].rect)).toBe(0);
  });

  it('treats a kept manual label as an obstacle for an untuned label', () => {
    const labels: LabelBox[] = [
      {
        id: 'manual',
        anchor: { x: 60, y: 60 },
        width: 60,
        height: 14,
        kind: 'component',
        priority: 0,
        manualRect: { x: 200, y: 200, width: 60, height: 14 },
      },
      {
        // Untuned label whose own node sits right next to the kept rect.
        id: 'untuned',
        anchor: { x: 205, y: 222 },
        width: 60,
        height: 14,
        kind: 'component',
        priority: 1,
      },
    ];
    const placed = autoPlaceLabels(labels, [], bounds, config);
    const manual = placed.find((p) => p.id === 'manual');
    const untuned = placed.find((p) => p.id === 'untuned');
    expect(rectsOverlapArea(manual.rect, untuned.rect)).toBe(0);
  });

  it('biases a re-placed manual label toward the authored position', () => {
    // Authored position collides with a marker; the cleared slot should be the
    // one nearest the authored rect, not the default NE slot near the anchor.
    const obstacles: Obstacle[] = [{ type: 'circle', x: 220, y: 66, radius: 12 }];
    const labels: LabelBox[] = [
      {
        id: 'm',
        anchor: { x: 100, y: 300 },
        width: 40,
        height: 12,
        kind: 'component',
        priority: 0,
        manualRect: { x: 200, y: 60, width: 40, height: 12 },
      },
    ];
    const [placed] = autoPlaceLabels(labels, obstacles, bounds, config);
    const center = {
      x: placed.rect.x + placed.rect.width / 2,
      y: placed.rect.y + placed.rect.height / 2,
    };
    // Far closer to the authored area (220, 66) than to the anchor (100, 300).
    const distToManual = Math.hypot(center.x - 220, center.y - 66);
    const distToAnchor = Math.hypot(center.x - 100, center.y - 300);
    expect(distToManual).toBeLessThan(distToAnchor);
  });

  it('still places untuned labels and is deterministic with manual labels present', () => {
    const labels: LabelBox[] = [
      {
        id: 'm',
        anchor: { x: 100, y: 100 },
        width: 40,
        height: 12,
        kind: 'component',
        priority: 0,
        manualRect: { x: 200, y: 60, width: 40, height: 12 },
      },
      { id: 'u', anchor: { x: 150, y: 150 }, width: 40, height: 12, kind: 'component', priority: 1 },
    ];
    expect(autoPlaceLabels(labels, [], bounds, config)).toEqual(
      autoPlaceLabels(labels, [], bounds, config)
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts -t "autoPlaceLabels with manual labels"`
Expected: FAIL — manual labels are not kept; `placed.rect` does not equal the authored `manualRect`.

- [ ] **Step 3: Rewrite `autoPlaceLabels` with the keep/pool flow**

In `wardleyLabelPlacement.ts`, replace the entire `autoPlaceLabels` function (from `export const autoPlaceLabels = (` to its closing `};`) with this version. The greedy machinery (`constraintOf`, `placeAll`, refinement) is the same; what's new is the partition, the kept-label handling, the `keptObstacleRects` threaded into `placeAll`/`constraintOf`, and the per-label `preferredCenter` passed to `bestCandidate`:

```ts
/**
 * Place every label to minimise overlap with other labels, node markers, the
 * chart boundary, and link lines.
 *
 * A label carrying a `manualRect` (an author-specified `label [x, y]`) is kept
 * exactly when that position is collision-free; such kept labels become fixed
 * obstacles. Every other label — untuned labels and manual labels whose
 * authored position collided — is placed by the greedy algorithm: most
 * constrained first, ties broken by `priority`, then a refinement pass. A
 * re-placed manual label is biased back toward its authored position.
 * Pure and deterministic.
 */
export const autoPlaceLabels = (
  labels: LabelBox[],
  obstacles: Obstacle[],
  bounds: Rect,
  config: PlacementConfig
): PlacedLabel[] => {
  if (labels.length === 0) {
    return [];
  }
  if (config.slotDistances.length === 0) {
    throw new Error('autoPlaceLabels: config.slotDistances must be non-empty');
  }
  const { slotDistances, leaderThreshold, refinementCount } = config;

  // Partition manual (author-positioned) labels from untuned ones.
  const manualLabels = labels.filter((label) => label.manualRect !== undefined);
  const untunedLabels = labels.filter((label) => label.manualRect === undefined);

  // Classify each manual label: kept (collision-free) or pooled (collided).
  const finalById = new Map<string, PlacedLabel>();
  const keptObstacleRects: Rect[] = [];
  const pooledManual: LabelBox[] = [];
  for (const label of manualLabels) {
    const otherManualRects = manualLabels
      .filter((other) => other.id !== label.id)
      .map((other) => other.manualRect!);
    if (isManualLabelKept(label, obstacles, bounds, otherManualRects)) {
      finalById.set(label.id, {
        id: label.id,
        rect: { ...label.manualRect! },
        anchor: label.anchor,
        needsLeader: false,
      });
      keptObstacleRects.push(label.manualRect!);
    } else {
      pooledManual.push(label);
    }
  }

  // The pool: untuned labels + manual labels whose authored position collided.
  const pool = [...untunedLabels, ...pooledManual];

  // The preferred-position bias for a pooled label (manual labels only).
  const preferredCenterOf = (label: LabelBox): Point | undefined => {
    if (!label.manualRect) {
      return undefined;
    }
    return {
      x: label.manualRect.x + label.manualRect.width / 2,
      y: label.manualRect.y + label.manualRect.height / 2,
    };
  };

  if (pool.length > 0) {
    // Count candidates under obstacle/boundary pressure (incl. kept rects);
    // labels with more such candidates have fewer good options, placed first.
    const constraintOf = (label: LabelBox): number => {
      const candidates = generateCandidates(label, slotDistances);
      let blocked = 0;
      for (const candidate of candidates) {
        if (
          scoreCandidate(candidate, obstacles, bounds, label.anchor, keptObstacleRects) > 1
        ) {
          blocked++;
        }
      }
      return blocked;
    };

    // Sort most-constrained first; ties broken deterministically by priority.
    const order = [...pool].sort((a, b) => {
      const diff = constraintOf(b) - constraintOf(a);
      return diff !== 0 ? diff : a.priority - b.priority;
    });

    const placed = new Map<string, { label: LabelBox; scored: Scored }>();

    const placeAll = (sequence: LabelBox[]) => {
      for (const label of sequence) {
        // Obstacle rects = kept manual labels + every other pooled label placed.
        const others: Rect[] = [...keptObstacleRects];
        for (const [id, entry] of placed) {
          if (id !== label.id) {
            others.push(entry.scored.candidate.rect);
          }
        }
        const candidates = generateCandidates(label, slotDistances);
        const preferred = preferredCenterOf(label);
        let best: Scored | undefined;
        for (const candidate of candidates) {
          const score = scoreCandidate(
            candidate,
            obstacles,
            bounds,
            label.anchor,
            others,
            preferred
          );
          if (best === undefined || score < best.score) {
            best = { candidate, score };
          }
        }
        placed.set(label.id, { label, scored: best! });
      }
    };

    // First pass.
    placeAll(order);

    // Refinement pass: re-place the worst-scoring labels against the full layout.
    const worst = [...placed.values()]
      .sort((a, b) => b.scored.score - a.scored.score)
      .slice(0, Math.max(0, refinementCount))
      .map((entry) => entry.label);
    placeAll(worst);

    for (const label of pool) {
      const { scored } = placed.get(label.id)!;
      const center = {
        x: scored.candidate.rect.x + scored.candidate.rect.width / 2,
        y: scored.candidate.rect.y + scored.candidate.rect.height / 2,
      };
      const dist = Math.hypot(center.x - label.anchor.x, center.y - label.anchor.y);
      finalById.set(label.id, {
        id: label.id,
        rect: scored.candidate.rect,
        anchor: label.anchor,
        needsLeader: dist > leaderThreshold,
      });
    }
  }

  // Output in the original input order for stable consumers.
  return labels.map((label) => finalById.get(label.id)!);
};
```

Note: this replacement removes the old standalone `bestCandidate` call inside `placeAll` and inlines the candidate loop so the per-label `preferredCenter` can be threaded through. The module-level `bestCandidate` function is now unused — see the next step.

- [ ] **Step 4: Remove the now-unused `bestCandidate` helper**

The old `placeAll` called the module-level `bestCandidate`; the new `placeAll` inlines that loop to pass `preferredCenter`. Delete the now-dead `bestCandidate` function (the `const bestCandidate = (label, distances, obstacles, bounds, placedRects): Scored => { ... };` block). Leave the `Scored` interface — the new code still uses it.

If a lint/type check reports `bestCandidate` is still referenced somewhere, do not delete it — instead report that as a surprise. (It should only have been used inside the old `placeAll`.)

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — the new `autoPlaceLabels with manual labels` tests pass AND all pre-existing `autoPlaceLabels` tests still pass (untuned-only maps behave exactly as before).

- [ ] **Step 6: Type-check**

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS — no errors (in particular, no "unused `bestCandidate`" error).

- [ ] **Step 7: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): keep collision-free manual labels in autoPlaceLabels"
```

---

## Task 5: Renderer — compute `manualRect` for manually-offset nodes

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts`

- [ ] **Step 1: Locate the node-label collection in `buildPlacement`**

Read `buildPlacement` in `wardleyRenderer.ts`. It loops over `input.nodes`; for each node with a position it calls `estimateLabelBox(node.label, input.fontSize)` and pushes a `LabelBox` with `id: \`node:${node.id}\``, `anchor`, `width`, `height`, `kind`, `priority`. This task adds a `manualRect` to that pushed object when the node has a manual offset.

The relevant facts about the node data (from `wardleyBuilder.ts` / `wardleyRenderer.ts`):
- `node.labelOffsetX` and `node.labelOffsetY` are `number | undefined`. The parser sets them as a pair from `label [x, y]`, so a manually-positioned label has both defined.
- In the legacy (flag-off) render path, a component label is drawn with `text-anchor: start` at `x = pos.x + labelOffsetX`, `y = pos.y + labelOffsetY` (baseline `auto`). An anchor label is drawn with `text-anchor: middle`, `dominant-baseline: middle` at the same `x`/`y`.

- [ ] **Step 2: Add `manualRect` computation in the node loop**

In `buildPlacement`, inside the `for (const node of input.nodes)` loop, after `const box = estimateLabelBox(node.label, input.fontSize);` and before the `labels.push({...})` call, add:

```ts
    // When the author specified `label [x, y]`, record that authored position
    // as a rect so autoPlaceLabels can keep it if it is collision-free.
    let manualRect: { x: number; y: number; width: number; height: number } | undefined;
    if (node.labelOffsetX !== undefined && node.labelOffsetY !== undefined) {
      const textX = pos.x + node.labelOffsetX;
      const textY = pos.y + node.labelOffsetY;
      if (node.className === 'anchor') {
        // Anchor labels render text-anchor:middle, dominant-baseline:middle.
        manualRect = {
          x: textX - box.width / 2,
          y: textY - box.height / 2,
          width: box.width,
          height: box.height,
        };
      } else {
        // Component labels render text-anchor:start, baseline:auto (text sits
        // above the baseline), so the rect's left edge is textX and its top
        // edge is one line-height above textY.
        manualRect = {
          x: textX,
          y: textY - box.height,
          width: box.width,
          height: box.height,
        };
      }
    }
```

- [ ] **Step 3: Attach `manualRect` to the pushed `LabelBox`**

In the same loop, add `manualRect` to the object passed to `labels.push({...})`. The existing push looks like:

```ts
    labels.push({
      id: `node:${node.id}`,
      anchor: { x: pos.x, y: pos.y },
      width: box.width,
      height: box.height,
      kind: node.className === 'anchor' ? 'anchor' : 'component',
      priority: priority++,
    });
```

Change it to add the field (a `manualRect` of `undefined` is harmless — the field is optional):

```ts
    labels.push({
      id: `node:${node.id}`,
      anchor: { x: pos.x, y: pos.y },
      width: box.width,
      height: box.height,
      kind: node.className === 'anchor' ? 'anchor' : 'component',
      priority: priority++,
      manualRect,
    });
```

- [ ] **Step 4: Type-check**

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS — no new wardley errors. (`manualRect` is an optional `Rect` on `LabelBox`; the inline object literal is structurally a `Rect`.)

- [ ] **Step 5: Run the wardley unit tests**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley`
Expected: PASS — all wardley specs.

- [ ] **Step 6: Visually verify**

Run: `pnpm --filter mermaid run dev`, then render (via `mermaid.initialize({ 'wardley-beta': { autoPlaceLabels: true } })`) a map that has:
- a component with a sensible `label [x, y]` that overlaps nothing — confirm the label stays exactly at the authored offset;
- a component with a `label [x, y]` placed on top of another node — confirm that label moves off the collision but stays in the same general area;
- components with no `label` — confirm they are auto-placed as before.
With the flag off, confirm the map renders identically to before this change.

- [ ] **Step 7: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts
git commit -m "feat(wardley): pass authored label positions into autoplacement"
```

---

## Task 6: Cypress visual regression test

**Files:**
- Modify: `cypress/integration/rendering/wardley/wardley.spec.js`

- [ ] **Step 1: Add the test**

In `cypress/integration/rendering/wardley/wardley.spec.js`, after the existing `should render dense map labels without overlap when autoPlaceLabels is enabled` test (and before the final closing `});` of the `describe` block), add:

```js
  it('should keep collision-free manual labels when autoPlaceLabels is enabled', () => {
    // Mix of three cases: a component with a sensible manual label that should
    // be kept untouched, a component whose manual label is dropped on top of
    // another node (should be re-placed), and untuned components (auto-placed).
    imgSnapshotTest(
      `
wardley-beta
title Manual Label Mix
size [1100, 800]

component Good Manual Label [0.30, 0.55] label [40, -20]
component Colliding Manual [0.55, 0.60] label [-90, 2]
component Crowded Node A [0.52, 0.62]
component Crowded Node B [0.56, 0.585]
component Crowded Node C [0.53, 0.59]
component Untuned Component [0.70, 0.45]

Good Manual Label -> Crowded Node A
Colliding Manual -> Crowded Node B
Crowded Node C -> Untuned Component
      `,
      { 'wardley-beta': { autoPlaceLabels: true } }
    );
  });
```

- [ ] **Step 2: Confirm the spec file is valid JS**

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS (the Cypress JS file is not type-checked by this, but confirm nothing else broke).

Do NOT run the full Cypress suite — it needs a browser and an image-snapshot baseline; that is a CI/manual step. Just confirm the new `it` block follows the same shape as the other tests in the file and no existing test was modified.

- [ ] **Step 3: Run the wardley unit tests once more**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley`
Expected: PASS — all wardley specs.

- [ ] **Step 4: Commit**

```bash
git add cypress/integration/rendering/wardley/wardley.spec.js
git commit -m "test(wardley): visual test for kept vs re-placed manual labels"
```

---

## Self-Review Notes

- **Spec coverage:** `manualRect` field + collision-free keep → Task 1 + Task 4. Kept label is an obstacle → Task 4 (`keptObstacleRects` threaded into `placeAll` and `constraintOf`). Collision check scope (static obstacles + other manual rects, own marker excluded) → Task 3 (`isManualLabelKept`). Two mutually-overlapping manual labels both re-placed → Task 3 logic + Task 4 test. Colliding manual label biased toward authored position → Task 1 (`preferredCenter` in `scoreCandidate`) + Task 2 (manual candidate slot) + Task 4 (`preferredCenterOf` wired into `placeAll`). Renderer computes `manualRect` → Task 5. Determinism → Task 4 test. Flag-off unchanged → Task 5 Step 6 + the renderer change only adds an optional field. Testing → Tasks 1-4 (unit), Task 6 (visual).
- **Type consistency:** `LabelBox.manualRect` is `Rect | undefined`, defined in Task 1 and consumed in Tasks 2-5. `isManualLabelKept(label, obstacles, bounds, otherManualRects)` defined in Task 3, called in Task 4 with exactly those argument types. `scoreCandidate`'s new 6th parameter `preferredCenter?: Point` defined in Task 1, passed in Task 4. `Scored` interface is retained (Task 4 Step 4 keeps it); `bestCandidate` is removed (Task 4 Step 4).
- **Known deviation flagged:** Task 4 removes the module-level `bestCandidate` helper because the new `placeAll` inlines the candidate loop to thread `preferredCenter`. Task 4 Step 4 calls this out explicitly with a guard against an unexpected remaining reference.
