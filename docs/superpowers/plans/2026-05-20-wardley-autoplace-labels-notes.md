# Wardley autoplace-labels: width-estimator validation notes

Date: 2026-05-20
Branch: feat/wardley-ecosystem-attitudes

## The estimator approach

`estimateLabelBox` in `wardleyLabelPlacement.ts` computes a bounding box for a text label without a DOM round-trip:

```
width  = max(1, text.length) * fontSize * GLYPH_WIDTH_FACTOR   // GLYPH_WIDTH_FACTOR = 0.6
height = fontSize * 1.2
```

The factor `0.6` is a commonly-used approximation for the average glyph advance width of a proportional sans-serif font at a given `em` size. It over-estimates narrow characters (i, l) and under-estimates wide ones (m, w), but for typical label strings the average is close enough to prevent visual overlap.

## Why estimation was chosen over getBBox()

- **Determinism**: the estimator produces identical results across environments, making unit tests reproducible without a browser.
- **No DOM round-trip**: `getBBox()` requires a live SVG element attached to a rendered document. Calling it during placement forces a layout flush and cannot be used in a pure module.
- **Unit-testable**: `wardleyLabelPlacement.spec.ts` exercises placement logic fully with Vitest (no browser). A `getBBox`-based path would require Cypress or jsdom with SVG support.
- **Sufficient accuracy**: for collision-avoidance the estimator only needs to be within roughly one character width of the true size to make a meaningful improvement. A 10–20 % error is acceptable.

## Validation status

The empirical comparison of estimated vs. measured (`getBBox`) width was not performed as a separate measurement step. The validation artifact is the `autoPlaceLabels` snapshot test in `cypress/integration/rendering/wardley/wardley.spec.js`.

That test renders a deliberately dense map (seven closely-spaced components, six links) with `autoPlaceLabels: true`. When the snapshot is first recorded in CI:

- If labels are well-separated with small leader lines, the estimator is working correctly.
- If labels visibly overlap or have excessive whitespace gaps, the `GLYPH_WIDTH_FACTOR` constant needs tuning (see below).

The snapshot must be **visually reviewed** when it is first recorded. It cannot be validated programmatically by this test alone.

## Fallback / tuning lever

If the recorded snapshot shows overlap or large gaps, the documented remedy is:

1. **Tune `GLYPH_WIDTH_FACTOR`** in `wardleyLabelPlacement.ts`. Measure a representative sample of labels with `getBBox()` in a browser console and compute `observedWidth / (text.length * fontSize)`. Use the mean ratio as the new factor.
2. **Add a `getBBox`-based measurement path** in `wardleyRenderer.ts`: after the SVG is attached to the DOM, call `getBBox()` on each label element and feed the real sizes back into a second placement pass. This is more accurate but requires two render passes and cannot be unit-tested without a browser.

The single constant `GLYPH_WIDTH_FACTOR` in `wardleyLabelPlacement.ts` is the primary lever. No other changes to the placement algorithm should be needed for typical label sets.
