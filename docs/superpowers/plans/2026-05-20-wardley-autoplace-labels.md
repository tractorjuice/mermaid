# Wardley Autoplace Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add opt-in automatic label placement to the `wardley-beta` diagram so component, anchor, link, and annotation labels avoid overlapping each other, node markers, the chart boundary, and link lines.

**Architecture:** A new pure, DOM-free module (`wardleyLabelPlacement.ts`) generates candidate positions for each label, scores them against obstacles, and places labels greedily in a deterministic order with one refinement pass. Text width is estimated from a character-width model rather than measured in the DOM, keeping the whole feature unit-testable. The renderer (`wardleyRenderer.ts`) collects label/obstacle data, calls the module when the config flag is on, applies the returned positions, and draws leader lines for labels moved past a distance threshold.

**Tech Stack:** TypeScript, D3 (SVG rendering), Vitest (unit tests), Cypress (visual regression).

---

## File Structure

| File | Responsibility |
|------|----------------|
| `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts` | NEW — pure geometry, width estimator, candidate generation, scoring, greedy placement. The unit-tested core. |
| `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts` | NEW — unit tests for the core. |
| `packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts` | MODIFY — collect labels/obstacles, call placement, apply positions, draw leader lines. |
| `packages/mermaid/src/schemas/config.schema.yaml` | MODIFY — add `autoPlaceLabels` to `WardleyDiagramConfig` (def at line 2503). |
| `packages/mermaid/src/config.type.ts` | REGENERATED from schema — not hand-edited. |
| `cypress/integration/rendering/wardley/wardley.spec.js` | MODIFY — add a dense example map rendered with `autoPlaceLabels: true`. |

**Test runner:** `npx vitest run <path>` from the repo root (`/workspaces/mermaid`).

---

## Task 1: Add the `autoPlaceLabels` config flag

**Files:**
- Modify: `packages/mermaid/src/schemas/config.schema.yaml:2503-2554` (the `WardleyDiagramConfig` def)
- Regenerate: `packages/mermaid/src/config.type.ts`
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts:67-81` (the `getConfigValues` function)

- [ ] **Step 1: Add the property to the schema**

In `config.schema.yaml`, inside `WardleyDiagramConfig` → `properties`, after the `showGrid` block (around line 2552), add:

```yaml
      autoPlaceLabels:
        description: |
          When `true`, component, anchor, link, and annotation labels are
          automatically repositioned to avoid overlapping each other, node
          markers, the chart boundary, and link lines. Manual `label [x, y]`
          offsets are ignored while this is enabled.
        type: boolean
        default: false
```

- [ ] **Step 2: Regenerate the config type**

Run: `pnpm --filter mermaid run types:build-config`
Expected: `packages/mermaid/src/config.type.ts` updates; `WardleyDiagramConfig` (around line 1785) gains `autoPlaceLabels?: boolean`.

- [ ] **Step 3: Verify the type generation is consistent**

Run: `pnpm --filter mermaid run types:verify-config`
Expected: PASS, no diff reported.

- [ ] **Step 4: Expose the flag in the renderer**

In `wardleyRenderer.ts`, inside `getConfigValues()`, add a line to the returned object after `showGrid`:

```ts
    showGrid: wardleyConfig?.showGrid ?? false,
    autoPlaceLabels: wardleyConfig?.autoPlaceLabels ?? false,
    useMaxWidth: wardleyConfig?.useMaxWidth ?? true,
```

- [ ] **Step 5: Verify the renderer still type-checks**

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS (no errors referencing wardley files).

- [ ] **Step 6: Commit**

```bash
git add packages/mermaid/src/schemas/config.schema.yaml packages/mermaid/src/config.type.ts packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts
git commit -m "feat(wardley): add autoPlaceLabels config flag"
```

---

## Task 2: Geometry primitives and width estimator

**Files:**
- Create: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

- [ ] **Step 1: Write the failing test**

Create `wardleyLabelPlacement.spec.ts`:

```ts
import { describe, it, expect } from 'vitest';
import {
  estimateLabelBox,
  rectsOverlapArea,
  circleRectOverlap,
  segmentIntersectsRect,
  areaOutsideBounds,
} from './wardleyLabelPlacement.js';

describe('geometry primitives', () => {
  it('estimates a label box from text and font size', () => {
    const box = estimateLabelBox('Tea', 10);
    expect(box.width).toBeGreaterThan(0);
    expect(box.height).toBeGreaterThan(10);
    // longer text -> wider box
    expect(estimateLabelBox('Tea Shop', 10).width).toBeGreaterThan(box.width);
  });

  it('computes overlap area of two rects', () => {
    const a = { x: 0, y: 0, width: 10, height: 10 };
    const b = { x: 5, y: 5, width: 10, height: 10 };
    expect(rectsOverlapArea(a, b)).toBe(25);
    expect(rectsOverlapArea(a, { x: 100, y: 100, width: 5, height: 5 })).toBe(0);
  });

  it('detects circle/rect overlap', () => {
    const rect = { x: 0, y: 0, width: 10, height: 10 };
    expect(circleRectOverlap({ x: 5, y: 5, radius: 3 }, rect)).toBe(true);
    expect(circleRectOverlap({ x: 100, y: 100, radius: 3 }, rect)).toBe(false);
  });

  it('detects segment/rect intersection', () => {
    const rect = { x: 0, y: 0, width: 10, height: 10 };
    expect(segmentIntersectsRect({ x1: -5, y1: 5, x2: 15, y2: 5 }, rect)).toBe(true);
    expect(segmentIntersectsRect({ x1: -5, y1: -5, x2: -1, y2: -1 }, rect)).toBe(false);
  });

  it('computes area of a rect lying outside bounds', () => {
    const bounds = { x: 0, y: 0, width: 100, height: 100 };
    expect(areaOutsideBounds({ x: 10, y: 10, width: 10, height: 10 }, bounds)).toBe(0);
    // half outside on the left
    expect(areaOutsideBounds({ x: -5, y: 10, width: 10, height: 10 }, bounds)).toBe(50);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: FAIL — cannot resolve `./wardleyLabelPlacement.js`.

- [ ] **Step 3: Create the module with primitives**

Create `wardleyLabelPlacement.ts`:

```ts
export interface Point {
  x: number;
  y: number;
}

export interface Rect {
  x: number;
  y: number;
  width: number;
  height: number;
}

export interface Circle {
  x: number;
  y: number;
  radius: number;
}

export interface Segment {
  x1: number;
  y1: number;
  x2: number;
  y2: number;
}

/**
 * Estimate a label's bounding box from its text and font size.
 * Uses an average glyph-width factor for sans-serif fonts. This avoids a
 * DOM measurement round-trip and keeps placement deterministic and testable.
 */
const GLYPH_WIDTH_FACTOR = 0.6;
const LINE_HEIGHT_FACTOR = 1.2;

export const estimateLabelBox = (
  text: string,
  fontSize: number
): { width: number; height: number } => ({
  width: Math.max(1, text.length) * fontSize * GLYPH_WIDTH_FACTOR,
  height: fontSize * LINE_HEIGHT_FACTOR,
});

/** Area of intersection between two axis-aligned rects (0 if disjoint). */
export const rectsOverlapArea = (a: Rect, b: Rect): number => {
  const xOverlap = Math.max(
    0,
    Math.min(a.x + a.width, b.x + b.width) - Math.max(a.x, b.x)
  );
  const yOverlap = Math.max(
    0,
    Math.min(a.y + a.height, b.y + b.height) - Math.max(a.y, b.y)
  );
  return xOverlap * yOverlap;
};

/** True if a circle intersects (or touches) an axis-aligned rect. */
export const circleRectOverlap = (circle: Circle, rect: Rect): boolean => {
  const closestX = Math.max(rect.x, Math.min(circle.x, rect.x + rect.width));
  const closestY = Math.max(rect.y, Math.min(circle.y, rect.y + rect.height));
  const dx = circle.x - closestX;
  const dy = circle.y - closestY;
  return dx * dx + dy * dy <= circle.radius * circle.radius;
};

/** True if a line segment crosses or lies inside an axis-aligned rect. */
export const segmentIntersectsRect = (seg: Segment, rect: Rect): boolean => {
  const { x, y, width, height } = rect;
  // Endpoint inside the rect.
  const inside = (px: number, py: number) =>
    px >= x && px <= x + width && py >= y && py <= y + height;
  if (inside(seg.x1, seg.y1) || inside(seg.x2, seg.y2)) {
    return true;
  }
  // Segment/segment intersection against the four rect edges.
  const cross = (ax: number, ay: number, bx: number, by: number, cx: number, cy: number) =>
    (bx - ax) * (cy - ay) - (by - ay) * (cx - ax);
  const segIntersect = (
    p1: Point, p2: Point, p3: Point, p4: Point
  ): boolean => {
    const d1 = cross(p3.x, p3.y, p4.x, p4.y, p1.x, p1.y);
    const d2 = cross(p3.x, p3.y, p4.x, p4.y, p2.x, p2.y);
    const d3 = cross(p1.x, p1.y, p2.x, p2.y, p3.x, p3.y);
    const d4 = cross(p1.x, p1.y, p2.x, p2.y, p4.x, p4.y);
    return ((d1 > 0) !== (d2 > 0)) && ((d3 > 0) !== (d4 > 0));
  };
  const a = { x: seg.x1, y: seg.y1 };
  const b = { x: seg.x2, y: seg.y2 };
  const corners = [
    { x, y },
    { x: x + width, y },
    { x: x + width, y: y + height },
    { x, y: y + height },
  ];
  for (let i = 0; i < 4; i++) {
    if (segIntersect(a, b, corners[i], corners[(i + 1) % 4])) {
      return true;
    }
  }
  return false;
};

/** Area of `rect` that lies outside `bounds`. */
export const areaOutsideBounds = (rect: Rect, bounds: Rect): number => {
  const inside = rectsOverlapArea(rect, bounds);
  return rect.width * rect.height - inside;
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — all 5 tests green.

- [ ] **Step 5: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): add label-placement geometry primitives"
```

---

## Task 3: Candidate slot generation

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `wardleyLabelPlacement.spec.ts`:

```ts
import { generateCandidates } from './wardleyLabelPlacement.js';
import type { LabelBox } from './wardleyLabelPlacement.js';

describe('candidate generation', () => {
  const componentLabel: LabelBox = {
    id: 'a',
    anchor: { x: 100, y: 100 },
    width: 40,
    height: 12,
    kind: 'component',
    priority: 0,
  };

  it('generates 8 directions x 2 distances for a component label', () => {
    const candidates = generateCandidates(componentLabel, [12, 24]);
    expect(candidates).toHaveLength(16);
  });

  it('returns rects sized to the label', () => {
    const [first] = generateCandidates(componentLabel, [12, 24]);
    expect(first.rect.width).toBe(40);
    expect(first.rect.height).toBe(12);
  });

  it('places link-label candidates on both perpendicular sides', () => {
    const linkLabel: LabelBox = {
      id: 'l',
      anchor: { x: 50, y: 50 },
      width: 20,
      height: 12,
      kind: 'link',
      priority: 0,
      preferredOffset: { x: 0, y: 1 },
    };
    const candidates = generateCandidates(linkLabel, [10, 20]);
    expect(candidates.length).toBeGreaterThanOrEqual(4);
    // candidates straddle the anchor on the y axis
    expect(candidates.some((c) => c.rect.y < 50)).toBe(true);
    expect(candidates.some((c) => c.rect.y > 50)).toBe(true);
  });

  it('is deterministic — same input yields identical candidates', () => {
    expect(generateCandidates(componentLabel, [12, 24])).toEqual(
      generateCandidates(componentLabel, [12, 24])
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: FAIL — `generateCandidates` is not exported.

- [ ] **Step 3: Implement candidate generation**

Append to `wardleyLabelPlacement.ts`:

```ts
export type LabelKind = 'component' | 'anchor' | 'link' | 'annotation';

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
}

export interface Candidate {
  rect: Rect;
  /** Unit direction from anchor to the candidate's center. */
  direction: Point;
  /** Distance from the anchor to the candidate's center. */
  distance: number;
}

// 8 compass directions, diagonals normalized to unit length.
const SQRT1_2 = Math.SQRT1_2;
const COMPASS: Point[] = [
  { x: 1, y: 0 }, // E
  { x: SQRT1_2, y: -SQRT1_2 }, // NE
  { x: 0, y: -1 }, // N
  { x: -SQRT1_2, y: -SQRT1_2 }, // NW
  { x: -1, y: 0 }, // W
  { x: -SQRT1_2, y: SQRT1_2 }, // SW
  { x: 0, y: 1 }, // S
  { x: SQRT1_2, y: SQRT1_2 }, // SE
];

const rectAround = (center: Point, width: number, height: number): Rect => ({
  x: center.x - width / 2,
  y: center.y - height / 2,
  width,
  height,
});

/**
 * Generate the set of candidate positions for a label.
 * Component / anchor / annotation labels use the 8 compass directions at each
 * supplied distance. Link labels use only the perpendicular axis (both sides).
 */
export const generateCandidates = (
  label: LabelBox,
  distances: number[]
): Candidate[] => {
  const candidates: Candidate[] = [];
  const directions =
    label.kind === 'link'
      ? perpendicularDirections(label.preferredOffset)
      : COMPASS;
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
  return candidates;
};

/** The two unit vectors perpendicular to a link's direction hint. */
const perpendicularDirections = (hint?: Point): Point[] => {
  const h = hint ?? { x: 0, y: 1 };
  const len = Math.hypot(h.x, h.y) || 1;
  const ux = h.x / len;
  const uy = h.y / len;
  // Rotate +/-90 degrees.
  return [
    { x: -uy, y: ux },
    { x: uy, y: -ux },
  ];
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — all candidate tests green.

- [ ] **Step 5: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): add label candidate generation"
```

---

## Task 4: Candidate scoring

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `wardleyLabelPlacement.spec.ts`:

```ts
import { scoreCandidate } from './wardleyLabelPlacement.js';
import type { Obstacle } from './wardleyLabelPlacement.js';

describe('candidate scoring', () => {
  const bounds = { x: 0, y: 0, width: 200, height: 200 };
  const clearCandidate = {
    rect: { x: 50, y: 50, width: 30, height: 12 },
    direction: { x: SQRT1_2_TEST, y: -SQRT1_2_TEST },
    distance: 12,
  };

  it('scores a clear candidate near zero', () => {
    const score = scoreCandidate(clearCandidate, [], bounds, { x: 60, y: 80 });
    expect(score).toBeLessThan(1);
  });

  it('penalizes overlap with a placed label', () => {
    const placed: Rect[] = [{ x: 55, y: 52, width: 30, height: 12 }];
    const clear = scoreCandidate(clearCandidate, [], bounds, { x: 60, y: 80 });
    const overlapping = scoreCandidate(
      clearCandidate,
      [],
      bounds,
      { x: 60, y: 80 },
      placed
    );
    expect(overlapping).toBeGreaterThan(clear);
  });

  it('heavily penalizes a candidate outside bounds', () => {
    const offMap = {
      rect: { x: -20, y: 50, width: 30, height: 12 },
      direction: { x: -1, y: 0 },
      distance: 12,
    };
    const score = scoreCandidate(offMap, [], bounds, { x: 10, y: 56 });
    expect(score).toBeGreaterThan(1000);
  });

  it('penalizes overlap with a node marker obstacle', () => {
    const obstacles: Obstacle[] = [
      { type: 'circle', x: 65, y: 56, radius: 6 },
    ];
    const withMarker = scoreCandidate(
      clearCandidate,
      obstacles,
      bounds,
      { x: 60, y: 80 }
    );
    const withoutMarker = scoreCandidate(
      clearCandidate,
      [],
      bounds,
      { x: 60, y: 80 }
    );
    expect(withMarker).toBeGreaterThan(withoutMarker);
  });
});

const SQRT1_2_TEST = Math.SQRT1_2;
```

> Note: `SQRT1_2_TEST` is declared at the bottom but hoisted (`const` in module scope is fine for Vitest because the `describe` callback runs after the module body). If your linter objects, move the `const SQRT1_2_TEST = Math.SQRT1_2;` line above the `describe` block.

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: FAIL — `scoreCandidate` / `Obstacle` not exported.

- [ ] **Step 3: Implement scoring**

Append to `wardleyLabelPlacement.ts`:

```ts
export type Obstacle =
  | { type: 'circle'; x: number; y: number; radius: number }
  | { type: 'segment'; x1: number; y1: number; x2: number; y2: number };

// Penalty weights. Internal constants — deliberately not exposed as config.
const WEIGHT_LABEL_OVERLAP = 5;
const WEIGHT_MARKER_OVERLAP = 800;
const WEIGHT_OUT_OF_BOUNDS = 50;
const WEIGHT_LINK_CROSS = 120;
const WEIGHT_DISTANCE = 0.4;
const WEIGHT_DIRECTION = 6;

// Preferred direction: up-right (NE), matching the legacy default offset.
const PREFERRED_DIRECTION: Point = { x: Math.SQRT1_2, y: -Math.SQRT1_2 };

/**
 * Score a candidate position. Lower is better. A score of 0 is a perfect,
 * unobstructed placement in the preferred direction touching the anchor.
 */
export const scoreCandidate = (
  candidate: Candidate,
  obstacles: Obstacle[],
  bounds: Rect,
  anchor: Point,
  placedRects: Rect[] = []
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
  penalty += candidate.distance * WEIGHT_DISTANCE;

  // Direction deviation: 0 when aligned with preferred, up to 2 when opposite.
  const dot =
    candidate.direction.x * PREFERRED_DIRECTION.x +
    candidate.direction.y * PREFERRED_DIRECTION.y;
  penalty += (1 - dot) * WEIGHT_DIRECTION;

  return penalty;
};
```

> `anchor` is part of the signature for symmetry with later use and future leader-length scoring; the current body uses `candidate.distance`. Keep the parameter.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — all scoring tests green.

- [ ] **Step 5: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): add label candidate scoring"
```

---

## Task 5: Greedy placement with refinement pass

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts`
- Test: `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `wardleyLabelPlacement.spec.ts`:

```ts
import { autoPlaceLabels } from './wardleyLabelPlacement.js';
import type { PlacementConfig } from './wardleyLabelPlacement.js';

describe('autoPlaceLabels', () => {
  const bounds = { x: 0, y: 0, width: 400, height: 400 };
  const config: PlacementConfig = {
    slotDistances: [12, 24, 40],
    leaderThreshold: 30,
    refinementCount: 2,
  };

  it('returns one placed label per input label', () => {
    const labels: LabelBox[] = [
      { id: 'a', anchor: { x: 100, y: 100 }, width: 40, height: 12, kind: 'component', priority: 0 },
      { id: 'b', anchor: { x: 200, y: 200 }, width: 40, height: 12, kind: 'component', priority: 1 },
    ];
    const placed = autoPlaceLabels(labels, [], bounds, config);
    expect(placed).toHaveLength(2);
    expect(placed.map((p) => p.id).sort()).toEqual(['a', 'b']);
  });

  it('separates two labels sharing the same anchor', () => {
    const labels: LabelBox[] = [
      { id: 'a', anchor: { x: 200, y: 200 }, width: 50, height: 12, kind: 'component', priority: 0 },
      { id: 'b', anchor: { x: 200, y: 200 }, width: 50, height: 12, kind: 'component', priority: 1 },
    ];
    const placed = autoPlaceLabels(labels, [], bounds, config);
    const overlap = rectsOverlapArea(placed[0].rect, placed[1].rect);
    expect(overlap).toBe(0);
  });

  it('keeps every label inside bounds', () => {
    const labels: LabelBox[] = [
      { id: 'corner', anchor: { x: 396, y: 4 }, width: 60, height: 12, kind: 'component', priority: 0 },
    ];
    const [placed] = autoPlaceLabels(labels, [], bounds, config);
    expect(areaOutsideBounds(placed.rect, bounds)).toBe(0);
  });

  it('flags labels moved past the leader threshold', () => {
    // Three labels at one point force at least one far placement.
    const labels: LabelBox[] = [0, 1, 2].map((i) => ({
      id: `n${i}`,
      anchor: { x: 200, y: 200 },
      width: 80,
      height: 12,
      kind: 'component' as const,
      priority: i,
    }));
    const placed = autoPlaceLabels(labels, [], bounds, config);
    expect(placed.some((p) => p.needsLeader)).toBe(true);
  });

  it('is deterministic — identical input yields identical output', () => {
    const labels: LabelBox[] = [
      { id: 'a', anchor: { x: 100, y: 100 }, width: 40, height: 12, kind: 'component', priority: 0 },
      { id: 'b', anchor: { x: 110, y: 105 }, width: 40, height: 12, kind: 'component', priority: 1 },
    ];
    expect(autoPlaceLabels(labels, [], bounds, config)).toEqual(
      autoPlaceLabels(labels, [], bounds, config)
    );
  });

  it('handles empty input without throwing', () => {
    expect(autoPlaceLabels([], [], bounds, config)).toEqual([]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: FAIL — `autoPlaceLabels` / `PlacementConfig` not exported.

- [ ] **Step 3: Implement greedy placement**

Append to `wardleyLabelPlacement.ts`:

```ts
export interface PlacementConfig {
  /** Candidate ring distances, in pixels, from anchor to label center. */
  slotDistances: number[];
  /** A label moved farther than this from its anchor gets `needsLeader`. */
  leaderThreshold: number;
  /** Number of worst-scoring labels re-placed in the refinement pass. */
  refinementCount: number;
}

export interface PlacedLabel {
  id: string;
  /** Final bounding box of the label. */
  rect: Rect;
  /** The anchor point the label belongs to (for drawing a leader line). */
  anchor: Point;
  /** True when the label is far enough from its anchor to warrant a leader. */
  needsLeader: boolean;
}

interface Scored {
  candidate: Candidate;
  score: number;
}

const bestCandidate = (
  label: LabelBox,
  distances: number[],
  obstacles: Obstacle[],
  bounds: Rect,
  placedRects: Rect[]
): Scored => {
  const candidates = generateCandidates(label, distances);
  let best: Scored | undefined;
  for (const candidate of candidates) {
    const score = scoreCandidate(candidate, obstacles, bounds, label.anchor, placedRects);
    if (best === undefined || score < best.score) {
      best = { candidate, score };
    }
  }
  // generateCandidates always returns >= 1 candidate for distances.length >= 1.
  return best as Scored;
};

/**
 * Place every label to minimise overlap with other labels, node markers, the
 * chart boundary, and link lines. Greedy: labels are placed most-constrained
 * first (fewest zero-penalty candidates), ties broken by `priority`. A single
 * refinement pass re-places the worst-scoring labels against the full layout.
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
  const { slotDistances, leaderThreshold, refinementCount } = config;

  // Constraint count = number of candidates with a non-trivial penalty.
  const constraintOf = (label: LabelBox): number => {
    const candidates = generateCandidates(label, slotDistances);
    let blocked = 0;
    for (const candidate of candidates) {
      if (scoreCandidate(candidate, obstacles, bounds, label.anchor, []) > 1) {
        blocked++;
      }
    }
    return blocked;
  };

  // Sort most-constrained first; ties broken deterministically by priority.
  const order = [...labels].sort((a, b) => {
    const diff = constraintOf(b) - constraintOf(a);
    return diff !== 0 ? diff : a.priority - b.priority;
  });

  const result = new Map<string, { label: LabelBox; scored: Scored }>();

  const placeAll = (sequence: LabelBox[]) => {
    for (const label of sequence) {
      // Obstacle rects = every other label already placed.
      const others: Rect[] = [];
      for (const [id, entry] of result) {
        if (id !== label.id) {
          others.push(entry.scored.candidate.rect);
        }
      }
      const scored = bestCandidate(label, slotDistances, obstacles, bounds, others);
      result.set(label.id, { label, scored });
    }
  };

  // First pass.
  placeAll(order);

  // Refinement pass: re-place the worst-scoring labels against the full layout.
  const worst = [...result.values()]
    .sort((a, b) => b.scored.score - a.scored.score)
    .slice(0, Math.max(0, refinementCount))
    .map((entry) => entry.label);
  placeAll(worst);

  // Build output in the original input order for stable consumers.
  return labels.map((label) => {
    const { scored } = result.get(label.id)!;
    const center = {
      x: scored.candidate.rect.x + scored.candidate.rect.width / 2,
      y: scored.candidate.rect.y + scored.candidate.rect.height / 2,
    };
    const dist = Math.hypot(center.x - label.anchor.x, center.y - label.anchor.y);
    return {
      id: label.id,
      rect: scored.candidate.rect,
      anchor: label.anchor,
      needsLeader: dist > leaderThreshold,
    };
  });
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts`
Expected: PASS — all `autoPlaceLabels` tests green.

- [ ] **Step 5: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.spec.ts
git commit -m "feat(wardley): add greedy label placement with refinement"
```

---

## Task 6: Renderer integration — collect labels and place component/anchor labels

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts`

This task builds the placement result once for all label kinds and applies it to component and anchor labels. Link labels and annotations are wired in Task 7; they are already included in the collected `LabelBox` list here so cross-kind overlap is avoided.

- [ ] **Step 1: Import the placement module**

At the top of `wardleyRenderer.ts`, after the existing `WardleyNode` import, add:

```ts
import {
  autoPlaceLabels,
  estimateLabelBox,
} from './wardleyLabelPlacement.js';
import type {
  LabelBox,
  Obstacle,
  PlacedLabel,
} from './wardleyLabelPlacement.js';
```

- [ ] **Step 2: Add a helper that builds the placement result**

In `wardleyRenderer.ts`, add a module-level function above `export const draw` (after `getConfigValues`). It collects every label box and obstacle, then calls `autoPlaceLabels`. Read the existing `draw` body first to confirm the names of: the `positions` map (`Map<string, {x,y,node}>`, ~line 378), the `data` object (`data.nodes`, `data.links`, `data.annotations`), and the `projectX`/`projectY` helpers (~line 194). The helper receives the values it needs as parameters:

```ts
interface PlacementInput {
  nodes: WardleyNode[];
  positions: Map<string, { x: number; y: number; node: WardleyNode }>;
  links: { source: string; target: string; label?: string }[];
  annotations: { x: number; y: number; text: string }[];
  projectX: (value: number) => number;
  projectY: (value: number) => number;
  bounds: { x: number; y: number; width: number; height: number };
  fontSize: number;
  nodeRadius: number;
}

const buildPlacement = (input: PlacementInput): Map<string, PlacedLabel> => {
  const labels: LabelBox[] = [];
  const obstacles: Obstacle[] = [];
  let priority = 0;

  // Component & anchor labels; node markers are obstacles.
  for (const node of input.nodes) {
    const pos = input.positions.get(node.id);
    if (!pos) {
      continue;
    }
    const box = estimateLabelBox(node.label, input.fontSize);
    labels.push({
      id: `node:${node.id}`,
      anchor: { x: pos.x, y: pos.y },
      width: box.width,
      height: box.height,
      kind: node.className === 'anchor' ? 'anchor' : 'component',
      priority: priority++,
    });
    obstacles.push({
      type: 'circle',
      x: pos.x,
      y: pos.y,
      radius: input.nodeRadius,
    });
  }

  // Link lines are obstacles; link labels are placed at the link midpoint.
  for (const link of input.links) {
    const a = input.positions.get(link.source);
    const b = input.positions.get(link.target);
    if (!a || !b) {
      continue;
    }
    obstacles.push({ type: 'segment', x1: a.x, y1: a.y, x2: b.x, y2: b.y });
    if (link.label) {
      const box = estimateLabelBox(link.label, input.fontSize);
      const mid = { x: (a.x + b.x) / 2, y: (a.y + b.y) / 2 };
      labels.push({
        id: `link:${link.source}->${link.target}`,
        anchor: mid,
        width: box.width,
        height: box.height,
        kind: 'link',
        priority: priority++,
        preferredOffset: { x: b.x - a.x, y: b.y - a.y },
      });
    }
  }

  // Annotation labels.
  for (let i = 0; i < input.annotations.length; i++) {
    const ann = input.annotations[i];
    const box = estimateLabelBox(ann.text, input.fontSize);
    labels.push({
      id: `annotation:${i}`,
      anchor: { x: input.projectX(ann.x), y: input.projectY(ann.y) },
      width: box.width,
      height: box.height,
      kind: 'annotation',
      priority: priority++,
    });
  }

  const placed = autoPlaceLabels(labels, obstacles, input.bounds, {
    slotDistances: [12, 22, 36, 54],
    leaderThreshold: 34,
    refinementCount: 3,
  });

  return new Map(placed.map((p) => [p.id, p]));
};
```

> Adjust the property names in `PlacementInput` (`link.source`, `link.label`, `ann.x`, `ann.text`, etc.) to match the actual shapes in `WardleyBuildResult` — open `wardleyBuilder.ts` to confirm. If link endpoints are stored as node objects rather than id strings, dereference accordingly.

- [ ] **Step 3: Call the helper inside `draw` when the flag is on**

In `draw`, after the `positions` map is fully populated and `projectX`/`projectY` are defined, add:

```ts
  const chartBounds = {
    x: configValues.padding,
    y: configValues.padding,
    width: configValues.width - configValues.padding * 2,
    height: configValues.height - configValues.padding * 2,
  };
  const placement = configValues.autoPlaceLabels
    ? buildPlacement({
        nodes: data.nodes,
        positions,
        links: data.links,
        annotations: data.annotations,
        projectX,
        projectY,
        bounds: chartBounds,
        fontSize: configValues.labelFontSize,
        nodeRadius: configValues.nodeRadius,
      })
    : undefined;
```

> Confirm `configValues.width`/`height` are the chart dimensions used for the boundary — if the renderer derives `chartWidth`/`chartHeight` separately (it does, near `projectX`), use those plus `padding` so `chartBounds` matches the drawable area exactly.

- [ ] **Step 4: Apply placement to component/anchor label `x` attribute**

In the `nodeEnter.append('text')` block (~line 912), change the `.attr('x', ...)` callback so it uses the placed rect when available. The placement rect is the label's bounding box; component labels render with `text-anchor: start` (use `rect.x`), anchor labels with `text-anchor: middle` (use the rect's horizontal center):

```ts
    .attr('x', (node) => {
      const pos = positions.get(node.id)!;
      const placed = placement?.get(`node:${node.id}`);
      if (placed) {
        return node.className === 'anchor'
          ? placed.rect.x + placed.rect.width / 2
          : placed.rect.x;
      }
      // --- existing static-offset logic unchanged below ---
      if (node.className === 'anchor') {
        return node.labelOffsetX !== undefined ? pos.x + node.labelOffsetX : pos.x;
      }
      let defaultOffset = configValues.nodeLabelOffset;
      if (node.sourceStrategy && node.labelOffsetX === undefined) {
        defaultOffset += 10;
      }
      const customOffset = node.labelOffsetX ?? defaultOffset;
      return pos.x + customOffset;
    })
```

- [ ] **Step 5: Apply placement to component/anchor label `y` attribute**

Similarly change the `.attr('y', ...)` callback. The rect is the full box; text `y` is the baseline. Component labels use `dominant-baseline: auto` — set `y` to the rect's vertical center and add a `dominant-baseline: middle` override for placed labels (next step). Anchor labels already use `dominant-baseline: middle`. So for placed labels return the rect's vertical center for both:

```ts
    .attr('y', (node) => {
      const pos = positions.get(node.id)!;
      const placed = placement?.get(`node:${node.id}`);
      if (placed) {
        return placed.rect.y + placed.rect.height / 2;
      }
      // --- existing static-offset logic unchanged below ---
      if (node.className === 'anchor') {
        return node.labelOffsetY !== undefined ? pos.y + node.labelOffsetY : pos.y - 3;
      }
      let defaultOffset = -configValues.nodeLabelOffset;
      if (node.sourceStrategy && node.labelOffsetY === undefined) {
        defaultOffset -= 10;
      }
      const customOffset = node.labelOffsetY ?? defaultOffset;
      return pos.y + customOffset;
    })
```

- [ ] **Step 6: Make the baseline consistent for placed labels**

Change the existing `.attr('dominant-baseline', ...)` on the node-label text so placed component labels also center vertically:

```ts
    .attr('dominant-baseline', (node) => {
      if (placement?.get(`node:${node.id}`)) {
        return 'middle';
      }
      return node.className === 'anchor' ? 'middle' : 'auto';
    })
```

- [ ] **Step 7: Build and visually verify**

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS.

Then build and open a test map manually:
Run: `pnpm --filter mermaid run dev`
Open the dev server, render a wardley-beta map with `%%{init: {'wardley-beta': {'autoPlaceLabels': true}}}%%` and several near-coincident components.
Expected: component/anchor labels no longer overlap each other or node markers; with the flag off, the map looks identical to before.

- [ ] **Step 8: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts
git commit -m "feat(wardley): autoplace component and anchor labels"
```

---

## Task 7: Renderer integration — link labels and annotations

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts`

The `placement` map from Task 6 already contains link and annotation entries. This task applies them in the link-label block (~lines 556-611) and the annotation block (~lines 957+).

- [ ] **Step 1: Apply placement to link labels**

Locate the link-label rendering block (text appended for `link.label`). Where it currently computes the midpoint + 8px perpendicular offset and sets `x`/`y`/rotation, branch on the placed result. When a placed entry exists, set `x`/`y` to the placed rect center, use `text-anchor: middle`, `dominant-baseline: middle`, and **skip the rotation** (auto-placed link labels read upright):

```ts
      .attr('x', (link) => {
        const placed = placement?.get(`link:${link.source}->${link.target}`);
        if (placed) {
          return placed.rect.x + placed.rect.width / 2;
        }
        return /* existing midpoint + perpendicular-offset x expression */;
      })
      .attr('y', (link) => {
        const placed = placement?.get(`link:${link.source}->${link.target}`);
        if (placed) {
          return placed.rect.y + placed.rect.height / 2;
        }
        return /* existing midpoint + perpendicular-offset y expression */;
      })
      .attr('transform', (link) => {
        const placed = placement?.get(`link:${link.source}->${link.target}`);
        if (placed) {
          return null; // no rotation for auto-placed link labels
        }
        return /* existing rotate(...) expression */;
      })
```

> Match the link key format to Task 6: `` `link:${link.source}->${link.target}` ``. Use the exact source/target property names from `wardleyBuilder.ts`.

- [ ] **Step 2: Apply placement to annotation labels**

Locate the annotation rendering block. Annotation text/boxes are currently clamped to bounds at a declared position. When a placed entry exists, set the text (and its surrounding box, if any) origin from the placed rect instead of the clamped declared position:

```ts
      .attr('x', (ann, i) => {
        const placed = placement?.get(`annotation:${i}`);
        if (placed) {
          return placed.rect.x + placed.rect.width / 2;
        }
        return /* existing clamped x */;
      })
      .attr('y', (ann, i) => {
        const placed = placement?.get(`annotation:${i}`);
        if (placed) {
          return placed.rect.y + placed.rect.height / 2;
        }
        return /* existing clamped y */;
      })
```

> If annotations render a background `rect` plus a `text`, offset the `rect` to `placed.rect.x` / `placed.rect.y` (top-left) and keep the text centered within it. The annotation `i` index must match the iteration order used in `buildPlacement` (Task 6, Step 2) — both iterate `data.annotations` in array order, so the indices align.

- [ ] **Step 3: Build and visually verify**

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS.

Run: `pnpm --filter mermaid run dev`
Render a map with `autoPlaceLabels: true` containing link labels and annotations near components.
Expected: link labels and annotations no longer overlap components or each other; flag off → unchanged.

- [ ] **Step 4: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts
git commit -m "feat(wardley): autoplace link and annotation labels"
```

---

## Task 8: Draw leader lines

**Files:**
- Modify: `packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts`

- [ ] **Step 1: Add a leader-line render pass**

In `draw`, after all label text has been appended (so leaders sit beneath labels but above the grid — append to the same root group, before the text group, or set a low opacity), add a pass that draws a thin line from each anchor to the near edge of each placed label flagged `needsLeader`. Insert after `placement` is computed and before the node-label text block:

```ts
  if (placement) {
    const leaderGroup = root.append('g').attr('class', 'wardley-leader-lines');
    for (const placed of placement.values()) {
      if (!placed.needsLeader) {
        continue;
      }
      const cx = placed.rect.x + placed.rect.width / 2;
      const cy = placed.rect.y + placed.rect.height / 2;
      // Clamp the label end to the rect edge facing the anchor.
      const dx = placed.anchor.x - cx;
      const dy = placed.anchor.y - cy;
      const halfW = placed.rect.width / 2;
      const halfH = placed.rect.height / 2;
      const scale =
        Math.abs(dx) * halfH > Math.abs(dy) * halfW
          ? halfW / (Math.abs(dx) || 1)
          : halfH / (Math.abs(dy) || 1);
      const edgeX = cx + dx * scale;
      const edgeY = cy + dy * scale;
      leaderGroup
        .append('line')
        .attr('x1', placed.anchor.x)
        .attr('y1', placed.anchor.y)
        .attr('x2', edgeX)
        .attr('y2', edgeY)
        .attr('class', 'wardley-leader-line');
    }
  }
```

> Ensure `root` is the same selection the rest of the diagram is appended to. Append `leaderGroup` before the node group so leader lines render underneath node markers and labels.

- [ ] **Step 2: Style the leader line**

In `packages/mermaid/src/diagrams/wardley/styles.ts`, add a rule for `.wardley-leader-line` consistent with the existing themed CSS (use a muted stroke — reuse the grid or link color variable already present):

```ts
  .wardley-leader-line {
    stroke: ${theme.gridColor};
    stroke-width: 0.75px;
  }
```

> Match the actual structure of `styles.ts` — it is a function returning a template string keyed off theme values. Follow the existing selector formatting in that file.

- [ ] **Step 3: Build and visually verify**

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS.

Run: `pnpm --filter mermaid run dev`
Render a dense map with `autoPlaceLabels: true` where some labels are pushed far from their nodes.
Expected: thin connector lines join far-placed labels to their nodes; close labels have no leader.

- [ ] **Step 4: Commit**

```bash
git add packages/mermaid/src/diagrams/wardley/wardleyRenderer.ts packages/mermaid/src/diagrams/wardley/styles.ts
git commit -m "feat(wardley): draw leader lines for far-placed labels"
```

---

## Task 9: Width-estimator validation and Cypress visual test

**Files:**
- Modify: `cypress/integration/rendering/wardley/wardley.spec.js`
- Create: `docs/superpowers/plans/2026-05-20-wardley-autoplace-labels-notes.md` (validation evidence)

- [ ] **Step 1: Validate the width estimator against real measurements**

Run: `pnpm --filter mermaid run dev`
Render two or three existing wardley example maps with `autoPlaceLabels: true`. In the browser devtools console, for each `.wardley-node-label` text element compare `el.getBBox().width` against `text.length * 10 * 0.6` (the estimator with `fontSize: 10`).

Record the results in `docs/superpowers/plans/2026-05-20-wardley-autoplace-labels-notes.md`: the per-label estimated-vs-measured widths and whether any visible overlap or excessive gap resulted.

**Decision gate:**
- If estimation error produces no visible overlaps/gaps → keep the estimator; note this in the file.
- If it does → adjust `GLYPH_WIDTH_FACTOR` in `wardleyLabelPlacement.ts` to the observed average ratio, re-test, and record the new value. Only if adjustment is insufficient, add a `getBBox`-based measurement path in the renderer (pass measured widths into `buildPlacement` instead of calling `estimateLabelBox`) and note the rationale.

Commit the notes file:

```bash
git add docs/superpowers/plans/2026-05-20-wardley-autoplace-labels-notes.md
git commit -m "docs(wardley): record label-width estimation validation"
```

(If `GLYPH_WIDTH_FACTOR` was changed, include `wardleyLabelPlacement.ts` in this commit.)

- [ ] **Step 2: Add a dense-map Cypress test with the flag on**

In `cypress/integration/rendering/wardley/wardley.spec.js`, add a new test following the existing `imgSnapshotTest` pattern used by the other cases in that file:

```js
it('should auto-place labels to avoid overlap', () => {
  imgSnapshotTest(
    `wardley-beta
    title Autoplace Dense Map
    component Cup of Tea [0.79, 0.61]
    component Cup [0.73, 0.78]
    component Tea [0.63, 0.81]
    component Hot Water [0.52, 0.80]
    component Water [0.38, 0.82]
    component Kettle [0.43, 0.35]
    component Power [0.10, 0.70]
    Cup of Tea -> Cup
    Cup of Tea -> Tea
    Cup of Tea -> Hot Water
    Hot Water -> Water
    Hot Water -> Kettle
    Kettle -> Power
    `,
    { wardley: { autoPlaceLabels: true } }
  );
});
```

> Confirm the config key the test harness expects — the schema/runtime key is `wardley-beta`. If `imgSnapshotTest`'s config object is passed straight to `mermaid.initialize`, use `'wardley-beta': { autoPlaceLabels: true }`. Match whatever the other wardley tests in this file already do for diagram config.

- [ ] **Step 3: Confirm an existing map is unaffected with the flag off**

Verify (visually, or by leaving the existing snapshot tests untouched) that the pre-existing wardley test cases in this file still pass without snapshot changes — the flag defaults to `false`, so their output must be byte-identical.

Run: `pnpm --filter mermaid exec tsc --noEmit -p ./tsconfig.json`
Expected: PASS.

- [ ] **Step 4: Run the wardley unit tests**

Run: `npx vitest run packages/mermaid/src/diagrams/wardley`
Expected: PASS — all wardley specs, including `wardleyLabelPlacement.spec.ts`.

- [ ] **Step 5: Commit**

```bash
git add cypress/integration/rendering/wardley/wardley.spec.js
git commit -m "test(wardley): add autoplace-labels visual regression test"
```

---

## Self-Review Notes

- **Spec coverage:** All-label scope → Task 6 (component/anchor) + Task 7 (link/annotation). Override manual offsets → Task 6 Steps 4-5 bypass `labelOffsetX/Y` when a placed result exists. Opt-in flag → Task 1. Obstacle set (labels, markers, boundary, link lines) → Task 4 scoring + Task 6 obstacle collection. Greedy algorithm + refinement → Task 5. Leader lines past threshold → Task 5 (`needsLeader`) + Task 8 (drawing). Estimated width + validation → Task 2 + Task 9 Step 1. Testing → Tasks 2-5 (unit), Task 9 (visual + estimator validation).
- **Type consistency:** `LabelBox`, `Obstacle`, `Candidate`, `PlacementConfig`, `PlacedLabel`, `Point`, `Rect`, `Circle`, `Segment` defined in Task 2-5 and consumed unchanged in Tasks 6-8. Label id scheme (`node:`, `link:`, `annotation:`) is identical in `buildPlacement` (Task 6) and all consumers (Tasks 6-8).
- **Known follow-up flagged in the plan:** `PlacementInput` field names and `WardleyBuildResult` shapes (link source/target, annotation fields) must be confirmed against `wardleyBuilder.ts` during Task 6 — the plan calls this out explicitly rather than guessing.
