# Wardley Tidy-Labels — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Cross-repo note:** This plan is executed across three repos owned by `tractorjuice` — `wardley-maps-mermaid` (Tasks 1–5), `arc-kit` (Task 6), `wardleymap_math_model` (Task 7). None are in the mermaid workspace; clone each where you run its tasks. Tasks 1–5 are sequential; Tasks 6 and 7 are independent of each other and both depend only on Task 4.

**Goal:** A `tidy` tool that rewrites a `wardley-beta` `.mmd` with computed `label [x, y]` offsets so labels don't overlap, plus a Claude Code hook in the two skill repos that runs it automatically whenever a wardley map is written.

**Architecture:** The tidy tool reuses mermaid's pure `wardleyLabelPlacement` engine (vendored as compiled JS), projects map coords to pixels exactly as the mermaid renderer does, runs `autoPlaceLabels`, and inverts the result back into `label [x, y]` DSL offsets. Existing authored labels are passed in as `manualRect` so collision-free ones are kept untouched (minimal diffs). The hook is a `PostToolUse` wrapper that detects wardley `.mmd` writes and runs the tool.

**Tech Stack:** Node.js 18+ ESM (`.mjs`), `node:test` + `node:assert` (stdlib — `wardley-maps-mermaid/tools/` has no npm dependencies), Claude Code hooks.

**Decision locked:** consuming repos invoke the tool via `npx github:tractorjuice/wardley-maps-mermaid tidy <file>` (one source of truth). Task 4 adds the `bin`. If `wardley-maps-mermaid` cannot carry a `bin`, the fallback is vendoring `tools/` into each consumer — note this and stop for guidance if so.

---

## File Structure

| Repo | File | Responsibility |
|------|------|----------------|
| `wardley-maps-mermaid` | `tools/vendor/wardleyLabelPlacement.js` | Vendored compiled placement engine (Task 1) |
| `wardley-maps-mermaid` | `tools/vendor/PROVENANCE.md` | Records the mermaid source commit + re-sync command |
| `wardley-maps-mermaid` | `tools/tidy.mjs` | `tidyMap(text)` transform + CLI entry (Tasks 2, 3) |
| `wardley-maps-mermaid` | `tools/tidy.test.mjs` | Unit tests for `tidyMap` (Task 2) |
| `wardley-maps-mermaid` | `package.json` | `bin` entry for `npx` (Task 4) |
| `wardley-maps-mermaid` | `tools/regenerate.mjs` | Gains an optional tidy step (Task 5) |
| `arc-kit` | `tools/tidy-hook.mjs` + `.claude/settings.json` | Hook wrapper + registration (Task 6) |
| `wardleymap_math_model` | `tools/tidy-hook.mjs` + `.claude/settings.json` | Hook wrapper + registration (Task 7) |

**Reference:** the proven prototype is `tidy-labels-draft.mts` in the mermaid repo root (untracked scratch). `tools/tidy.mjs` is its productionised port — same algorithm, with the `manualRect` keep-good refinement added.

---

## Task 1: Vendor the placement engine into `wardley-maps-mermaid`

**Repo:** `wardley-maps-mermaid`
**Files:**
- Create: `tools/vendor/wardleyLabelPlacement.js` (compiled)
- Create: `tools/vendor/wardleyLabelPlacement.d.ts` (compiled)
- Create: `tools/vendor/PROVENANCE.md`

- [ ] **Step 1: Obtain the source module**

Get `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts` from the mermaid repo. Use the branch where the wardley autoplacement feature lives (`feat/wardley-ecosystem-attitudes`) or `develop` if it has merged. Record the exact commit SHA — you need it for PROVENANCE.md. Save the `.ts` file to a temp location, e.g. `/tmp/wardleyLabelPlacement.ts`.

- [ ] **Step 2: Compile it to standalone ESM JavaScript**

The module has zero imports, so a single-file `tsc` emit works with no tsconfig or bundler.

Run:
```bash
cd wardley-maps-mermaid
mkdir -p tools/vendor
npx -y typescript@5 tsc /tmp/wardleyLabelPlacement.ts \
  --module es2022 --target es2022 --declaration \
  --outDir tools/vendor
```
Expected: `tools/vendor/wardleyLabelPlacement.js` and `tools/vendor/wardleyLabelPlacement.d.ts` are created, no compile errors.

- [ ] **Step 3: Verify the compiled module imports and runs**

Run:
```bash
node --input-type=module -e "
import { estimateLabelBox, autoPlaceLabels } from './tools/vendor/wardleyLabelPlacement.js';
const b = estimateLabelBox('Hello', 10);
console.log('estimateLabelBox:', JSON.stringify(b));
const placed = autoPlaceLabels(
  [{ id: 'a', anchor: { x: 100, y: 100 }, width: 40, height: 12, kind: 'component', priority: 0 }],
  [], { x: 0, y: 0, width: 400, height: 400 },
  { slotDistances: [12, 24], leaderThreshold: 30, refinementCount: 1 }
);
console.log('autoPlaceLabels:', JSON.stringify(placed));
"
```
Expected: prints a box `{"width":...,"height":12}` and a placed-label array with one entry containing a `rect`. No errors.

- [ ] **Step 4: Write the provenance file**

Create `tools/vendor/PROVENANCE.md`:
```markdown
# Vendored module provenance

`wardleyLabelPlacement.js` / `.d.ts` are compiled from mermaid's pure
label-placement module. Do not hand-edit — re-sync instead.

- Source: https://github.com/mermaid-js/mermaid
- Path: packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts
- Commit: <PASTE THE EXACT SHA FROM TASK 1 STEP 1>

## Re-sync

    npx -y typescript@5 tsc /path/to/wardleyLabelPlacement.ts \
      --module es2022 --target es2022 --declaration --outDir tools/vendor

Then update the Commit line above.
```
Replace `<PASTE ...>` with the real SHA.

- [ ] **Step 5: Commit**

```bash
git add tools/vendor/
git commit -m "chore(tools): vendor wardley label-placement engine"
```

---

## Task 2: `tidyMap` — the core transform

**Repo:** `wardley-maps-mermaid`
**Files:**
- Create: `tools/tidy.mjs`
- Test: `tools/tidy.test.mjs`

- [ ] **Step 1: Write the failing tests**

Create `tools/tidy.test.mjs`:
```js
import test from 'node:test';
import assert from 'node:assert/strict';
import { tidyMap } from './tidy.mjs';

test('adds label offsets to untuned components', () => {
  const src = [
    'wardley-beta',
    'size [900, 600]',
    'component Alpha Component [0.55, 0.50]',
    'component Beta Component [0.55, 0.50]',
    '',
  ].join('\n');
  const { text, changed } = tidyMap(src);
  assert.match(text, /component Alpha Component \[0\.55, 0\.50\] label \[-?\d+, -?\d+\]/);
  assert.match(text, /component Beta Component \[0\.55, 0\.50\] label \[-?\d+, -?\d+\]/);
  assert.ok(changed >= 1, 'expected at least one line changed');
});

test('is idempotent — tidying a tidied map changes nothing further', () => {
  const src = [
    'wardley-beta',
    'size [900, 600]',
    'component Alpha Component [0.55, 0.50]',
    'component Beta Component [0.40, 0.65]',
    '',
  ].join('\n');
  const once = tidyMap(src).text;
  const twice = tidyMap(once).text;
  assert.equal(twice, once);
});

test('keeps a collision-free authored label unchanged', () => {
  const src = [
    'wardley-beta',
    'size [900, 600]',
    'component Lonely [0.50, 0.50] label [40, -20]',
    '',
  ].join('\n');
  const { text } = tidyMap(src);
  assert.match(text, /component Lonely \[0\.50, 0\.50\] label \[40, -20\]/);
});

test('leaves non-component lines verbatim', () => {
  const src = [
    'wardley-beta',
    'title My Map',
    'size [900, 600]',
    'component Solo [0.50, 0.50]',
    '',
  ].join('\n');
  const { text } = tidyMap(src);
  assert.match(text, /^wardley-beta$/m);
  assert.match(text, /^title My Map$/m);
  assert.match(text, /^size \[900, 600\]$/m);
});
```

- [ ] **Step 2: Run the tests, confirm they FAIL**

Run: `node --test tools/tidy.test.mjs`
Expected: FAIL — `tidy.mjs` does not exist / `tidyMap` is not exported.

- [ ] **Step 3: Implement `tools/tidy.mjs`**

Create `tools/tidy.mjs` with exactly this content:
```js
/**
 * tidy.mjs — tidy the labels of a mermaid `wardley-beta` map.
 *
 * Reads wardley-beta source, computes non-overlapping label positions with
 * mermaid's pure placement engine, and rewrites each `component` /
 * pipeline-child line with a computed `label [x, y]` pixel offset. Existing
 * authored labels are kept when collision-free (minimal diffs).
 *
 * Exports `tidyMap(text)`. Run directly as a CLI — see the bottom of the file.
 *
 * Parser scope (line-oriented, not the full langium grammar):
 *   HANDLED: `size [w,h]`; `component <name> [vis,evo] [label[..]] [decorator]`;
 *     `pipeline <parent> { ... }` with child `component <name> [evo] [label[..]]`;
 *     links (`A -> B`, `-.->`, ports, quoted labels) as obstacles.
 *   SKIPPED (left verbatim): anchors (no anchor label-offset in the grammar —
 *     parsed as obstacles only, never relabelled), annotations, notes,
 *     accelerators, attitudes, evolves, evolution-stage customisation.
 */
import { autoPlaceLabels, estimateLabelBox } from './vendor/wardleyLabelPlacement.js';

// mermaid wardley renderer constants (getConfigValues defaults).
const DEFAULT_WIDTH = 900;
const DEFAULT_HEIGHT = 600;
const PADDING = 48;
const NODE_RADIUS = 6;
const LABEL_FONT_SIZE = 10;
const SQUARE_SIZE = NODE_RADIUS * 1.6; // pipeline parent square side
const PIPELINE_BOX_HEIGHT = NODE_RADIUS * 4;
const PIPELINE_BOX_PADDING = 15;
// Same config buildPlacement passes to autoPlaceLabels.
const PLACEMENT_CONFIG = {
  slotDistances: [12, 22, 36, 54],
  leaderThreshold: 34,
  refinementCount: 3,
};
const DECORATORS = ['build', 'buy', 'outsource', 'market', 'ecosystem'];

// 0-1 -> 0-100; 0-100 stays. Mirrors wardleyParser.toPercent.
const toPercent = (v) => (v <= 1 ? v * 100 : v);

// Parse a `label [ -?INT , -?INT ]` token's numbers, or null.
const parseLabel = (s) => {
  const m = /label\s*\[\s*(-?\d+)\s*,\s*(-?\d+)\s*\]/.exec(s);
  return m ? { ox: Number(m[1]), oy: Number(m[2]) } : null;
};

/**
 * Tidy the labels of a wardley-beta map.
 * @param {string} mmdText
 * @returns {{ text: string, changed: number, total: number }}
 */
export const tidyMap = (mmdText) => {
  const lines = mmdText.split('\n');

  // ---- 1. line-oriented parse ----
  let width = DEFAULT_WIDTH;
  let height = DEFAULT_HEIGHT;
  /** @type {Array<{lineIndex:number,kind:string,name:string,vis:number,evo:number,
   *   manualOffset:?{ox:number,oy:number},decorator:?string,pipelineParent:?string,
   *   isPipelineParent:boolean}>} */
  const components = [];
  const links = [];
  const pipelineChildren = new Map(); // parent name -> [child names]
  let currentPipeline;

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i].trim();
    if (!line || line.startsWith('//')) {
      continue;
    }

    const sizeM = /^size\s*\[\s*(\d+(?:\.\d+)?)\s*,\s*(\d+(?:\.\d+)?)\s*\]/.exec(line);
    if (sizeM) {
      width = Number(sizeM[1]);
      height = Number(sizeM[2]);
      continue;
    }

    const pipeOpen = /^pipeline\s+(.+?)\s*\{/.exec(line);
    if (pipeOpen) {
      currentPipeline = pipeOpen[1].trim();
      pipelineChildren.set(currentPipeline, []);
      const parent = components.find((c) => c.name === currentPipeline);
      if (parent) {
        parent.isPipelineParent = true;
      }
      continue;
    }
    if (currentPipeline && line === '}') {
      currentPipeline = undefined;
      continue;
    }

    if (currentPipeline) {
      const childM =
        /^component\s+(.+?)\s*\[\s*(-?\d+(?:\.\d+)?)\s*\](\s*label\s*\[[^\]]*\])?/.exec(line);
      if (childM) {
        components.push({
          lineIndex: i,
          kind: 'pipeline-child',
          name: childM[1].trim(),
          vis: 0, // inherits parent's vis below
          evo: toPercent(Number(childM[2])),
          manualOffset: childM[3] ? parseLabel(childM[3]) : null,
          decorator: null,
          pipelineParent: currentPipeline,
          isPipelineParent: false,
        });
        pipelineChildren.get(currentPipeline).push(childM[1].trim());
        continue;
      }
    }

    const anchorM =
      /^anchor\s+(.+?)\s*\[\s*(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)\s*\]/.exec(line);
    if (anchorM) {
      components.push({
        lineIndex: i,
        kind: 'anchor',
        name: anchorM[1].trim(),
        vis: toPercent(Number(anchorM[2])),
        evo: toPercent(Number(anchorM[3])),
        manualOffset: null,
        decorator: null,
        pipelineParent: null,
        isPipelineParent: false,
      });
      continue;
    }

    const compM =
      /^component\s+(.+?)\s*\[\s*(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)\s*\](\s*label\s*\[[^\]]*\])?\s*(build|buy|outsource|market|ecosystem)?/.exec(
        line
      );
    if (compM) {
      components.push({
        lineIndex: i,
        kind: 'component',
        name: compM[1].trim(),
        vis: toPercent(Number(compM[2])),
        evo: toPercent(Number(compM[3])),
        manualOffset: compM[4] ? parseLabel(compM[4]) : null,
        decorator: DECORATORS.includes(compM[5]) ? compM[5] : null,
        pipelineParent: null,
        isPipelineParent: false,
      });
      continue;
    }

    const linkM = /^(.+?)\s*\+?[<>]*\s*-?\.?-+>?\s*\+?[<>]*\s*(.+?)(?:\s*'[^']*')?$/.exec(line);
    if (
      linkM &&
      /-+>|-\.-/.test(line) &&
      !/^(component|anchor|note|annotation|pipeline|evolve|title|size)\b/.test(line)
    ) {
      links.push({ source: linkM[1].trim(), target: linkM[2].trim() });
    }
  }

  for (const c of components) {
    if (c.kind === 'pipeline-child' && c.pipelineParent) {
      const parent = components.find((p) => p.name === c.pipelineParent);
      if (parent) {
        c.vis = parent.vis;
      }
    }
  }

  // ---- 2. projection (exact mermaid replica) ----
  const chartWidth = width - PADDING * 2;
  const chartHeight = height - PADDING * 2;
  const projectX = (v) => PADDING + (v / 100) * chartWidth;
  const projectY = (v) => height - PADDING - (v / 100) * chartHeight;

  const pos = new Map(); // name -> { x, y }
  for (const c of components) {
    pos.set(c.name, { x: projectX(c.evo), y: projectY(c.vis) });
  }

  // ---- pipeline pre-pass: reposition parent square + collect boxes ----
  const pipelineBoxes = [];
  for (const [parentName, childNames] of pipelineChildren) {
    let minX = Infinity;
    let maxX = -Infinity;
    let y = 0;
    for (const cn of childNames) {
      const p = pos.get(cn);
      if (p) {
        minX = Math.min(minX, p.x);
        maxX = Math.max(maxX, p.x);
        y = p.y;
      }
    }
    if (minX === Infinity) {
      continue;
    }
    const parentPos = pos.get(parentName);
    if (parentPos) {
      parentPos.x = (minX + maxX) / 2;
      parentPos.y = y - PIPELINE_BOX_HEIGHT / 2 - SQUARE_SIZE / 6;
    }
    pipelineBoxes.push({
      x: minX - PIPELINE_BOX_PADDING,
      y: y - PIPELINE_BOX_HEIGHT / 2,
      width: maxX - minX + PIPELINE_BOX_PADDING * 2,
      height: PIPELINE_BOX_HEIGHT,
    });
  }

  // ---- 3. build LabelBox[] + Obstacle[] (mirror buildPlacement) ----
  const markerRadius = (c) => {
    if (c.isPipelineParent) {
      return SQUARE_SIZE;
    }
    return c.decorator ? NODE_RADIUS * 2 : NODE_RADIUS;
  };

  const labels = [];
  const obstacles = [];
  let priority = 0;
  const boxByName = new Map(); // name -> estimated {width,height}

  for (const c of components) {
    const p = pos.get(c.name);
    const box = estimateLabelBox(c.name, LABEL_FONT_SIZE);
    boxByName.set(c.name, box);

    // Existing authored label -> manualRect, so a collision-free one is kept.
    // Component text-anchor:start/baseline:auto; anchor middle/middle.
    let manualRect;
    if (c.manualOffset) {
      const textX = p.x + c.manualOffset.ox;
      const textY = p.y + c.manualOffset.oy;
      manualRect =
        c.kind === 'anchor'
          ? { x: textX - box.width / 2, y: textY - box.height / 2, width: box.width, height: box.height }
          : { x: textX, y: textY - box.height, width: box.width, height: box.height };
    }

    labels.push({
      id: `node:${c.name}`,
      anchor: { x: p.x, y: p.y },
      width: box.width,
      height: box.height,
      kind: c.kind === 'anchor' ? 'anchor' : 'component',
      priority: priority++,
      preferredDirection: c.kind === 'pipeline-child' ? { x: 0, y: 1 } : undefined,
      manualRect,
    });
    obstacles.push({ type: 'circle', x: p.x, y: p.y, radius: markerRadius(c) });
  }

  for (const link of links) {
    const a = pos.get(link.source);
    const b = pos.get(link.target);
    if (a && b) {
      obstacles.push({ type: 'segment', x1: a.x, y1: a.y, x2: b.x, y2: b.y });
    }
  }
  for (const box of pipelineBoxes) {
    obstacles.push({ type: 'rect', x: box.x, y: box.y, width: box.width, height: box.height });
  }

  const bounds = { x: PADDING, y: PADDING, width: chartWidth, height: chartHeight };

  // ---- 4. run the engine ----
  const placed = autoPlaceLabels(labels, obstacles, bounds, PLACEMENT_CONFIG);
  const placedById = new Map(placed.map((pl) => [pl.id, pl]));

  // ---- 5. invert each placed rect -> wardley `label [ox, oy]` pixel offset ----
  const offsets = new Map(); // name -> { ox, oy }
  for (const c of components) {
    if (c.kind === 'anchor') {
      continue; // grammar has no anchor label-offset production
    }
    const pl = placedById.get(`node:${c.name}`);
    if (!pl) {
      continue;
    }
    const node = pos.get(c.name);
    const r = pl.rect;
    // Inverse of the manualRect construction above (component form).
    const ox = Math.round(r.x - node.x);
    const oy = Math.round(r.y + r.height - node.y);
    offsets.set(c.name, { ox, oy });
  }

  // ---- 6. rewrite ----
  const outLines = [...lines];
  let changed = 0;
  let total = 0;
  for (const c of components) {
    if (c.kind === 'anchor') {
      continue;
    }
    total++;
    const off = offsets.get(c.name);
    if (!off) {
      continue;
    }
    const token = `label [${off.ox}, ${off.oy}]`;
    const original = outLines[c.lineIndex];
    const next = c.manualOffset
      ? original.replace(/label\s*\[[^\]]*\]/, token)
      : original.replace(']', `] ${token}`);
    if (next !== original) {
      changed++;
    }
    outLines[c.lineIndex] = next;
  }

  return { text: outLines.join('\n'), changed, total };
};
```

- [ ] **Step 4: Run the tests, confirm they PASS**

Run: `node --test tools/tidy.test.mjs`
Expected: PASS — all 4 tests green.

- [ ] **Step 5: Commit**

```bash
git add tools/tidy.mjs tools/tidy.test.mjs
git commit -m "feat(tools): add tidyMap label-placement transform"
```

---

## Task 3: `tidy` CLI entry

**Repo:** `wardley-maps-mermaid`
**Files:**
- Modify: `tools/tidy.mjs` (append a CLI section)

- [ ] **Step 1: Append the CLI to `tools/tidy.mjs`**

Add this to the END of `tools/tidy.mjs` (after the `tidyMap` export). It runs only when the file is executed directly, not when imported:
```js
// ---- CLI ----
// Usage:
//   node tools/tidy.mjs <file.mmd>          rewrite the file in place
//   node tools/tidy.mjs --check <file.mmd>  exit 1 if tidying would change it
//   node tools/tidy.mjs --stdout <file.mmd> print the tidied map, do not write
const isMain = (() => {
  if (!process.argv[1]) {
    return false;
  }
  return import.meta.url === new URL(`file://${process.argv[1]}`).href;
})();

if (isMain) {
  const { readFileSync, writeFileSync } = await import('node:fs');
  const { resolve } = await import('node:path');
  const args = process.argv.slice(2);
  const check = args.includes('--check');
  const stdout = args.includes('--stdout');
  const file = args.find((a) => !a.startsWith('--'));
  if (!file) {
    console.error('usage: node tools/tidy.mjs [--check|--stdout] <file.mmd>');
    process.exit(2);
  }
  const path = resolve(file);
  const src = readFileSync(path, 'utf8');
  const { text, changed, total } = tidyMap(src);
  if (check) {
    if (text !== src) {
      console.error(`tidy: ${file} would change (${changed}/${total} labels)`);
      process.exit(1);
    }
    console.error(`tidy: ${file} already tidy`);
    process.exit(0);
  }
  if (stdout) {
    process.stdout.write(text);
    process.exit(0);
  }
  if (text !== src) {
    writeFileSync(path, text, 'utf8');
    console.error(`tidy: ${file} — ${changed}/${total} labels updated`);
  } else {
    console.error(`tidy: ${file} — already tidy`);
  }
}
```

- [ ] **Step 2: Verify the CLI on a real map**

Get a real map to test (any wardley-beta `.mmd`); a repo sample works. Run:
```bash
node tools/tidy.mjs --stdout <some-existing-map>.mmd | head -20
```
Expected: prints the start of the tidied map; `component` lines carry `label [x, y]` tokens. No crash.

Then check-mode on the tidied output is a no-op:
```bash
node tools/tidy.mjs --stdout <some-existing-map>.mmd > /tmp/tidied.mmd
node tools/tidy.mjs --check /tmp/tidied.mmd
```
Expected: exit 0, prints "already tidy" (idempotent).

- [ ] **Step 3: Confirm the import path still works as a module**

Run: `node --test tools/tidy.test.mjs`
Expected: PASS — the CLI block is guarded by `isMain`, so importing `tidyMap` from the test does not trigger it.

- [ ] **Step 4: Commit**

```bash
git add tools/tidy.mjs
git commit -m "feat(tools): add tidy CLI (default/--check/--stdout)"
```

---

## Task 4: Expose `tidy` as an `npx` bin

**Repo:** `wardley-maps-mermaid`
**Files:**
- Modify: `package.json`

- [ ] **Step 1: Inspect the current `package.json`**

Read `package.json`. Note whether it already has a `bin` field and what `name` / `type` it declares. The tool needs ESM — confirm `"type": "module"` is present or that `.mjs` is used (it is, so `.mjs` works regardless).

- [ ] **Step 2: Add a `bin` entry**

Add (or extend) the `bin` field in `package.json` so the package exposes a `wardley-tidy` command, and create a thin bin shim if the repo prefers shims over pointing `bin` straight at `tools/tidy.mjs`. Simplest — point `bin` directly at the CLI-capable `tools/tidy.mjs`:
```json
{
  "bin": {
    "wardley-tidy": "tools/tidy.mjs"
  }
}
```
Ensure `tools/tidy.mjs` starts with a shebang as its very first line so it is directly executable:
```js
#!/usr/bin/env node
```
Add that shebang line as line 1 of `tools/tidy.mjs` if not already present (above the existing block comment).

- [ ] **Step 3: Verify it runs via npx from a clean checkout location**

Run (from outside the repo, simulating a consumer):
```bash
npx --yes "github:tractorjuice/wardley-maps-mermaid" wardley-tidy --check <some-map>.mmd
```
Expected: the command resolves, runs, and prints a tidy status line. (If `wardley-maps-mermaid` is private, this needs auth — note that for the hook tasks; a vendored fallback is then required.)

If `npx github:` resolution fails for repo-structure reasons, STOP and report — the fallback (vendor `tools/` into each consumer) changes Tasks 6–7.

- [ ] **Step 4: Commit**

```bash
git add package.json tools/tidy.mjs
git commit -m "feat(tools): expose wardley-tidy as an npx bin"
```

---

## Task 5: Wire tidy into `regenerate.mjs`

**Repo:** `wardley-maps-mermaid`
**Files:**
- Modify: `tools/regenerate.mjs`

- [ ] **Step 1: Read `regenerate.mjs`**

Read `tools/regenerate.mjs` fully. Identify where it writes each generated `.mmd` (it converts OWM sources via `convert.mjs` and writes a sibling `.mmd`). Note the variable holding the generated mermaid text and the write call.

- [ ] **Step 2: Apply `tidyMap` to generated output**

Import `tidyMap` at the top of `regenerate.mjs`:
```js
import { tidyMap } from './tidy.mjs';
```
At the point where the generated `.mmd` text is about to be written, pass it through `tidyMap` first:
```js
// existing: const mermaid = owmToMermaid(owmContent, filename);
const tidied = tidyMap(mermaid + '\n').text;
// write `tidied` instead of `mermaid`
```
Match the exact existing variable names and write call — adapt the snippet to them. Keep the `--dry-run` flag behaviour intact (tidy in memory, still skip the write under dry-run).

- [ ] **Step 3: Regenerate and spot-check**

Run: `node tools/regenerate.mjs`
Expected: completes; the generated `.mmd` files now carry tidied `label [x, y]` offsets. Pick one dense map, render it in mermaid (or open in a viewer), and confirm labels do not overlap.

- [ ] **Step 4: Run the existing repo test/validation**

Run the repo's existing validation if present (the tools README mentions a Mermaid 11.15.0 smoke test and `test-fidelity.mjs`). Run whatever the repo documents, e.g.:
```bash
node tools/test-fidelity.mjs
```
Expected: passes — tidying changes only `label [..]` offsets, not components/anchors/links/coords, so fidelity is unaffected. If fidelity checks label offsets specifically and now flags churn, update the expectation to accept tidied offsets and note it in the commit.

- [ ] **Step 5: Commit**

```bash
git add tools/regenerate.mjs
git commit -m "feat(tools): tidy labels as part of regenerate"
```

- [ ] **Step 6: Commit the re-tidied maps**

```bash
git add '*.mmd'
git commit -m "chore: re-tidy generated wardley maps"
```

---

## Task 6: Tidy-labels hook in `arc-kit`

**Repo:** `arc-kit`
**Files:**
- Create: `tools/tidy-hook.mjs`
- Modify: `.claude/settings.json`
- Modify: the wardley skill's doc/`SKILL.md` (one line)

- [ ] **Step 1: Explore the repo**

Find: the existing `.claude/settings.json` (create it if absent), whether it already defines `hooks`, where the wardley skill lives (look under `.claude/skills/` or `skills/` for a wardley-related `SKILL.md`), and how generated wardley `.mmd` files are named/located.

- [ ] **Step 2: Create the hook wrapper `tools/tidy-hook.mjs`**

Create `tools/tidy-hook.mjs`:
```js
#!/usr/bin/env node
/**
 * Claude Code PostToolUse hook — tidies a wardley-beta map after it is written.
 *
 * Reads the hook payload on stdin, and if the tool wrote/edited a file that is
 * a wardley-beta map, runs `wardley-tidy` on it in place. Anything else is a
 * no-op. Always exits 0 (a hook failure must not block the tool).
 */
import { readFileSync } from 'node:fs';
import { execFileSync } from 'node:child_process';

const TIDY_PKG = 'github:tractorjuice/wardley-maps-mermaid';

const main = () => {
  let payload;
  try {
    payload = JSON.parse(readFileSync(0, 'utf8'));
  } catch {
    return; // no/garbled payload — nothing to do
  }
  const filePath =
    payload?.tool_input?.file_path ?? payload?.tool_input?.path ?? undefined;
  if (!filePath || !/\.(mmd|owm|md)$/i.test(filePath)) {
    return;
  }
  let content;
  try {
    content = readFileSync(filePath, 'utf8');
  } catch {
    return;
  }
  // Only act on actual wardley-beta maps.
  if (!/^\s*wardley-beta\b/m.test(content)) {
    return;
  }
  try {
    execFileSync('npx', ['--yes', TIDY_PKG, 'wardley-tidy', filePath], {
      stdio: 'ignore',
    });
  } catch {
    // Tidy failed — leave the file as-is, never block the write.
  }
};

main();
process.exit(0);
```

- [ ] **Step 3: Register the hook in `.claude/settings.json`**

Merge this into `.claude/settings.json` (preserve any existing keys/hooks):
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "node tools/tidy-hook.mjs" }
        ]
      }
    ]
  }
}
```
If `PostToolUse` already exists, append the new entry to its array rather than replacing it.

- [ ] **Step 4: Test the hook end to end**

Create a scratch wardley map with deliberately overlapping components:
```bash
cat > /tmp/hook-test.mmd <<'EOF'
wardley-beta
size [900, 600]
component Alpha Component [0.55, 0.50]
component Beta Component [0.55, 0.50]
EOF
node tools/tidy-hook.mjs <<'EOF'
{"tool_input":{"file_path":"/tmp/hook-test.mmd"}}
EOF
cat /tmp/hook-test.mmd
```
Expected: after the hook runs, both `component` lines in `/tmp/hook-test.mmd` carry `label [x, y]` offsets. Then a non-wardley file is untouched:
```bash
echo 'flowchart TD' > /tmp/not-wardley.mmd
node tools/tidy-hook.mjs <<'EOF'
{"tool_input":{"file_path":"/tmp/not-wardley.mmd"}}
EOF
cat /tmp/not-wardley.mmd
```
Expected: `/tmp/not-wardley.mmd` still reads exactly `flowchart TD` — unchanged.

- [ ] **Step 5: Note the hook in the wardley skill doc**

In the wardley skill's `SKILL.md` (found in Step 1), add one line under its behaviour notes:
> Wardley `.mmd` maps are auto-tidied on write by the `tidy-hook` (`PostToolUse`) — label positions are normalised so labels do not overlap.

- [ ] **Step 6: Commit**

```bash
git add tools/tidy-hook.mjs .claude/settings.json
git add <path-to-the-wardley-SKILL.md>
git commit -m "feat: auto-tidy wardley map labels via PostToolUse hook"
```

---

## Task 7: Tidy-labels hook in `wardleymap_math_model`

**Repo:** `wardleymap_math_model`
**Files:**
- Create: `tools/tidy-hook.mjs`
- Modify: `.claude/settings.json`
- Modify: the wardley skill's doc/`SKILL.md` (one line)

- [ ] **Step 1: Explore the repo**

Find: the existing `.claude/settings.json` (create if absent) and any existing `hooks`; where the wardley skill lives; how wardley `.mmd` files are produced/located. Confirm a `tools/` directory exists or create one.

- [ ] **Step 2: Create `tools/tidy-hook.mjs`**

Create `tools/tidy-hook.mjs` with EXACTLY the same content as Task 6 Step 2 (the file is repo-independent — it shells out to the published `wardley-tidy`). Copy it verbatim.

- [ ] **Step 3: Register the hook in `.claude/settings.json`**

Merge the same `hooks` block as Task 6 Step 3 into this repo's `.claude/settings.json`, appending to an existing `PostToolUse` array if present.

- [ ] **Step 4: Test the hook end to end**

Run the same two checks as Task 6 Step 4 (overlapping wardley map gets `label [x,y]` offsets; a `flowchart TD` file is left untouched). Expected: identical results.

- [ ] **Step 5: Note the hook in the wardley skill doc**

Add the same one-line note as Task 6 Step 5 to this repo's wardley skill `SKILL.md`.

- [ ] **Step 6: Commit**

```bash
git add tools/tidy-hook.mjs .claude/settings.json
git add <path-to-the-wardley-SKILL.md>
git commit -m "feat: auto-tidy wardley map labels via PostToolUse hook"
```

---

## Self-Review Notes

- **Coverage:** Vendor the engine → Task 1. The tidy transform (parse/project/place/invert/rewrite, keep-good via `manualRect`) → Task 2. CLI with `--check`/`--stdout` → Task 3. `npx` distribution → Task 4. `regenerate.mjs` integration → Task 5. The hook in both skill repos → Tasks 6, 7. Skill-doc note → Tasks 6/7 Step 5.
- **Type/name consistency:** `tidyMap(text) -> { text, changed, total }` defined in Task 2, used by the CLI in Task 3 and by `regenerate.mjs` in Task 5. The vendored module's `autoPlaceLabels`/`estimateLabelBox` (Task 1) are imported by `tidy.mjs` (Task 2). `wardley-tidy` bin name (Task 4) is used by `tidy-hook.mjs` (Tasks 6/7). `tools/tidy-hook.mjs` is byte-identical between Tasks 6 and 7 by design.
- **Known dependency:** Tasks 6–7 assume `npx github:tractorjuice/wardley-maps-mermaid wardley-tidy` resolves (Task 4 Step 3 verifies it). If that repo is private or `npx` can't resolve it, the fallback is vendoring `tools/tidy.mjs` + `tools/vendor/` into each consumer and changing the hook's `execFileSync` to call the local copy — Task 4 Step 3 says STOP and report if so.
- **Approximation noted in code:** `estimateLabelBox` is a character-width estimate, not real font metrics — the same estimate mermaid's auto-place path uses; documented in the `tidy.mjs` header.
