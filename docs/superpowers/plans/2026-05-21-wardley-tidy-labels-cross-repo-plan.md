# Wardley Tidy-Labels — Cross-Repo Plan

**Date:** 2026-05-21
**Status:** Plan + proven draft. Execution happens in `wardley-maps-mermaid`, `arc-kit`, and `wardleymap_math_model` — none are in the mermaid workspace.

## Goal

Give the wardley skills in **`arc-kit`** and **`wardleymap_math_model`** a hook that
automatically *tidies the labels* of any wardley map the skills produce: a label
that collides with another label, a node marker, a pipeline box, or the chart
edge is repositioned; labels the author already placed well are left alone.

## Approach: source-rewrite, not render-time

The tidy tool reads a `wardley-beta` `.mmd`, computes label positions, and
writes computed `label [x, y]` offsets **back into the source**. Consequences:

- The tidied `.mmd` renders cleanly in **any** mermaid version — the offsets are
  plain DSL. No dependency on the unreleased `autoPlaceLabels` feature, and no
  need to control `mermaid.initialize`.
- The map *file* becomes the clean artifact, which is what the skills commit.

This fully decouples the work from the mermaid release schedule.

## The engine

`mermaid`'s `packages/mermaid/src/diagrams/wardley/wardleyLabelPlacement.ts` is a
**pure, dependency-free** module (no DOM, no d3) — geometry helpers,
`estimateLabelBox`, `isManualLabelKept`, and `autoPlaceLabels`. A draft prototype
(`tidy-labels-draft.mts`, repo root, untracked) imported it unchanged and tidied
a real 24-node map first try. This module is the reusable engine.

It currently lives only on the `feat/wardley-ecosystem-attitudes` branch; vendor
it from there now, or re-sync once that branch merges.

## Repos and changes

| Repo | Change |
|------|--------|
| `wardley-maps-mermaid` | Vendor the pure module into `tools/`; add `tools/tidy.mjs`; optionally wire tidy into `regenerate.mjs`. |
| `arc-kit` | Add a Claude Code hook that runs tidy on wardley `.mmd` writes; note it in the wardley skill docs. |
| `wardleymap_math_model` | Same hook. |
| `mermaid` | No change. (The module is the source of the vendored copy.) |

---

## Phase 1 — Vendor the pure module into `wardley-maps-mermaid/tools/`

The `tools/` directory is "Node 18+, no npm dependencies, pure stdlib." The
placement module fits that ethos — it is one pure file, ~575 lines, zero imports.

1. Compile `wardleyLabelPlacement.ts` standalone to ESM JavaScript:
   `npx tsc wardleyLabelPlacement.ts --module es2022 --target es2022 --declaration --outDir tools/vendor/`
   It has no imports, so this is a clean single-file emit — no bundler.
2. Commit `tools/vendor/wardleyLabelPlacement.js` (+ `.d.ts`).
3. Add `tools/vendor/PROVENANCE.md`: the mermaid repo URL, the exact commit SHA
   the module was compiled from, and the re-sync command. This makes drift
   visible and the copy refreshable.

> Alternative considered — publishing the module as an npm package or a mermaid
> subpath export. Rejected for now: vendoring keeps `tools/` dependency-free and
> the user owns all the repos, so a publish/version dance adds nothing.

## Phase 2 — Build `tools/tidy.mjs`

The productionised version of the draft. It exports `tidyMap(mmdText, options)`
and runs as a CLI (`node tools/tidy.mjs <file.mmd>`), mirroring `convert.mjs`'s
shape.

**Algorithm (proven by the draft):**

1. **Parse** the `.mmd` line-by-line: `size [w,h]`; each `component`/`anchor`
   with `[a, b]` coords and any existing `label [ox, oy]`; `pipeline { ... }`
   blocks and their child components; links (`A -> B`) for obstacles.
2. **Project to pixels — must match `wardleyRenderer.ts` exactly:**
   - Canvas: `width` 900 / `height` 600 default, overridden by `size [w,h]`;
     `padding` 48; `nodeRadius` 6; `labelFontSize` 10.
   - `chartWidth = width - padding*2`, `chartHeight = height - padding*2`.
   - `projectX(v) = padding + (v/100)*chartWidth`
   - `projectY(v) = height - padding - (v/100)*chartHeight`  *(Y inverted)*
   - `[a, b]` is `[visibility, evolution]` → **a → Y, b → X**. Pipeline children
     are `component name [evolution]` (single value) and inherit the parent's Y.
   - Coords accept 0–1 or 0–100: `toPercent(v) = v <= 1 ? v*100 : v`.
3. **Build `LabelBox[]` and `Obstacle[]`** mirroring `buildPlacement`:
   - One `LabelBox` per node; size via `estimateLabelBox(text, 10)`.
   - Set `manualRect` from any existing `label [ox, oy]` (so the keep-good logic
     runs — see below).
   - `preferredDirection: { x:0, y:1 }` (South) for pipeline children.
   - Obstacles: node marker circles, link segments, pipeline box rects
     (`height = nodeRadius*4 = 24`, horizontal padding 15; reposition the
     parent square to the child mid-x).
4. **Call `autoPlaceLabels(labels, obstacles, bounds, config)`** with the same
   config `buildPlacement` uses: `slotDistances:[12,22,36,54]`,
   `leaderThreshold:34`, `refinementCount:3`.
   Because each existing label is passed as a `manualRect`, `autoPlaceLabels`
   **keeps collision-free authored labels untouched** and only re-places the
   colliding ones — so a tidy produces a minimal diff and respects author work.
5. **Invert each `PlacedLabel.rect` to a `label [ox, oy]` pixel offset:**
   - Component (text-anchor:start, baseline:auto):
     `ox = round(rect.x - nodeX)` ; `oy = round(rect.y + rect.height - nodeY)`
   - Anchor (text-anchor:middle, baseline:middle):
     `ox = round(rect.x + rect.width/2 - nodeX)` ; `oy = round(rect.y + rect.height/2 - nodeY)`
6. **Rewrite** the `.mmd`: add/replace `label [ox, oy]` on each `component`
   line; everything else byte-for-byte verbatim.

**CLI modes:**
- default — rewrite the file in place (or to `<name>.tidied.mmd`).
- `--check` — exit non-zero if tidying *would* change the file (for CI / the hook
  to detect already-clean maps and skip).
- `--stdout` — print the tidied map without writing (useful for the hook).

**Improvements over the draft (the draft re-tidied everything and hard-coded
defaults):**
- Honour collision-free authored labels via `manualRect` (step 4) — minimal diffs.
- Read `wardley-beta` config overrides if a map sets `padding`/`nodeRadius`/etc.
- Leave anchor lines unchanged — the wardley grammar has no anchor label-offset
  production; anchors are obstacles only.

**Known approximation (documented, accepted):** `estimateLabelBox` uses
`length*fontSize*0.6` not real font metrics, so a tidied map rendered with real
text has minor horizontal drift. This is the same estimate the renderer's
auto-place path uses; acceptable for a tidy.

## Phase 3 — Wire tidy into `regenerate.mjs` (optional)

`regenerate.mjs` walks the repo `.owm` sources and emits `.mmd`. Add a tidy step
so the pipeline is `.owm → convert → .mmd → tidy`, and re-tidy the existing 147
`.mmd` files once. Keep tidy callable standalone too (the hook needs that).

## Phase 4 — The hook in `arc-kit` and `wardleymap_math_model`

A Claude Code `PostToolUse` hook in each repo's `.claude/settings.json`, matching
`Write|Edit`, running a small wrapper that:
1. Reads the hook payload (the written file path).
2. Skips unless the file is a wardley map (`.mmd`/`.owm`-shaped *and* its content
   begins with / contains `wardley-beta`).
3. Runs `tidy.mjs` on it (in place).

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

`tidy-hook.mjs` is the wrapper that does the wardley-map detection and calls
`tidy.mjs`. So whenever a wardley skill writes a map, its labels are tidied with
no skill change required.

**How the consuming repos get the tool** — open decision (see below).

**Note on staleness:** a `PostToolUse` hook rewrites the file the agent just
wrote, so the agent's cached view goes stale. For a label tidy that is harmless
(the skill rarely re-reads the map it just emitted). If it matters, the
alternative is a explicit final step in each wardley skill ("run tidy") instead
of a hook — but the user asked for a hook.

Also add one line to each repo's wardley skill doc: "Maps are auto-tidied on
write by the tidy-labels hook."

---

## Open decisions

1. **How `arc-kit` / `wardleymap_math_model` obtain `tidy.mjs` + the vendored
   module.** Options:
   - **a.** `wardley-maps-mermaid` becomes installable; the hook runs
     `npx github:tractorjuice/wardley-maps-mermaid tidy <file>`.
   - **b.** Vendor `tools/` (tidy + the module) into each consuming repo.
   - **c.** Git submodule of `wardley-maps-mermaid` in each repo.
   Recommendation: **a** if `wardley-maps-mermaid` can carry a `bin` entry —
   one source of truth, `npx` handles distribution. Otherwise **b**.
2. **Vendor the module now (from the feature branch) or after it merges to
   mermaid `develop`.** Provenance file makes either fine.
3. **Hook vs explicit skill step.** Plan assumes hook (per request); note the
   staleness trade-off above.

## Testing

- `tools/tidy.mjs` — unit-test the parse → project → invert round-trip on a
  small synthetic map; assert offsets are sane integers, idempotence (`tidy`
  of an already-tidied map is a no-op under `--check`).
- Re-tidy the 147 repo `.mmd` files; spot-check a dense one renders without
  label overlap.
- The vendored module keeps its own behaviour — it is the mermaid unit-tested
  code, unchanged.

## Reference — the proven draft

`tidy-labels-draft.mts` (mermaid repo root, untracked scratch) is the working
proof: it imports the real `wardleyLabelPlacement.ts`, tidied
`education-equity-infrastructure.mmd`, and produced sane offsets
(e.g. `infrastructure` `label [-84,-11]` → `[-42,-48]`; pipeline child
`renewables` → `[-30, 28]`, positive `oy` confirming the South bias). It is the
template for `tools/tidy.mjs`.
