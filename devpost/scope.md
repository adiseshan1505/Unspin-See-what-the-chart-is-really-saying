---
doc: scope
status: draft
---

# Unspin

Upload a bar chart screenshot. Unspin extracts the numbers, computes Tufte's Lie Factor, flags the axis trick, and puts an honest redraw next to the original.

## The Unique Kernel
**The Lie Factor turns "this chart feels off" into a number.** Lie Factor = (size of the effect shown in the graphic) / (size of the effect in the data). It answers one question: does the picture exaggerate the data, or match it?

The point isn't fraud detection. Charts can be accurate and still give a false impression. Unspin makes that visible so people look harder before they draw conclusions. The original and the honest redraw sit side by side, built from the same numbers, so the argument is made by the pictures and not by a paragraph of AI explanation.

## Who It's For
Retail investors reading charts from finfluencers and pitch decks. They see a bar chart like "Our Fund's 2025 Returns vs. S&P 500" and take the bar heights at face value. They don't check whether the y-axis starts at zero, and in practice they never do.

## The Core Loop
1. Upload a screenshot of a bar chart.
2. See the extracted numbers in a table and correct any value that was read wrong.
3. See the verdict: the Lie Factor, which trick was found, and what it does to the picture.
4. Compare the original with Unspin's honest redraw.

They come back whenever a chart looks too dramatic to be true.

## Inspiration & Identity
- Edward Tufte's Lie Factor, from *The Visual Display of Quantitative Information*.
- The verdict uses red, warning-style emphasis when a chart is misleading.
- The tone is "look closer," not "gotcha." The side-by-side comparison should land in under a second.

## Why This Matters to the Learner
"What interested me is the Lie Factor concept … the real game here." The project brings data visualization and data science together "to convince the world that appearance might be deceptive," and that "we just need to be more careful before we make any inferences."

## What "Working" Looks Like
**Demo chart:** "Our Fund's 2025 Returns vs. S&P 500." Two bars, Fund 14% and S&P 500 11%, with the y-axis starting at 9 instead of 0.
- The drawn bar heights are 5 vs 2, so the bars look 2.5x apart. The real relative gap is about 27%.
- Computed Lie Factor = 1.5 / 0.273 ≈ **5.5**.

**The minute on screen:**
1. Upload the chart. A loading state reads "Reading chart...".
2. **Extracted data table:** "My Fund: 14%" and "S&P 500: 11%", each with an edit control.
3. **Verdict**, front and center, in warning styling. For example: "⚠ Misleading — Lie Factor [computed]. Y-axis starts at 9 instead of 0."
4. **Side by side:** the original upload on the left, exactly as uploaded, and the honest redraw on the right, with the same data and the axis starting at 0.

**The "oh" moment:** in the original, the fund's bar towers over the S&P's. In the redraw, the two bars are nearly the same height, a minor edge rather than a dramatic one.

**It's a tool, not a slideshow:** the pipeline has to work on any bar chart within the boundary, not just the demo chart.

## The POC Boundary
- **Input:** a single uploaded image of a bar chart with 2–4 bars.
- **Extraction:** bar labels and values, shown in an editable table. Edits feed the calculation and the redraw.
- **Detection:** two tricks, a **truncated y-axis** and an **inverted y-axis**.
- **Lie Factor:** computed from the extracted values and the chart's axis, and shown in the verdict.
- **Honest redraw:** the same data with an honest axis, shown next to the original.

## Later
- Line charts, pie charts
- Dual-axis detection
- 3D or area distortion
- Detecting cherry-picked date ranges
- Multi-series charts, or more than 4 bars
- Any input besides an uploaded image: URLs, a browser extension
- Batch upload
- Saving, history, accounts

## Explicitly Cut
Nothing is cut outright. Everything not in the POC is deferred to Later.

## Inverted Axis: Direction Is Separate from Size
The Lie Factor measures **size**, computed on magnitudes. An inverted axis is flagged separately as a **direction** reversal. So a chart can be accurate in size but reversed in direction, and it's still misleading. Learner's example verdict: "Axis inverted. The bar that looks taller (My Fund) is actually the smaller value. Lie Factor: 1.1 (magnitude is accurate, but direction is reversed)."

## Open Questions for the PRD
- What Unspin shows when a chart is honest (Lie Factor ≈ 1), and when the image isn't a readable bar chart.
- Whether horizontal bar charts count as "bar charts" for the POC.
