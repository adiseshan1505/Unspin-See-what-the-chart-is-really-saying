---
doc: prd
status: approved
---

# Unspin — Product Requirements

**Unspin — See what the chart is really saying.** A single-page tool for retail investors reading charts from finfluencers and pitch decks. Upload a bar chart screenshot. Unspin extracts the numbers, computes Tufte's Lie Factor, flags a truncated or inverted y-axis, and shows an honest redraw next to the original.
Source: `scope.md > The Unique Kernel`, `scope.md > Who It's For`.

## The Core Journey
Source: `scope.md > The Core Loop`, `scope.md > What "Working" Looks Like`.

1. **Arrive.** The user opens Unspin and sees one page: masthead, one line of context, and a large upload box. No login and no navigation.
2. **Upload.** They drag a screenshot onto the box, or click it to open the system file picker.
3. **Reading.** As soon as the file lands, a "Reading chart..." state appears.
4. **Results** appear in three stacked sections:
   1. **Extracted data table:** the axis start and each bar's value, all editable.
   2. **Verdict stamp:** MISLEADING or LOOKS HONEST, the Lie Factor, and what the trick does.
   3. **Side by side:** the original image on the left, the honest redraw on the right.
5. **Correct.** If anything was misread, the user edits it in the table. The verdict and the redraw recompute live.
6. **Success:** the user sees, in under a second, that the same numbers tell a different story when drawn honestly.
7. **Again.** The user can upload another chart without reloading the page.

## Screens and Layout
One page, no other screens. Top to bottom:

- **Masthead:** "Unspin — See what the chart is really saying."
- **Context line:** "Upload a bar chart screenshot and we'll check if the axis is lying to you."
- **Upload box:** large, centered, dashed border. Text: "Drag a chart image here, or click to browse."
- **Expectation note** below the box, small and muted: "Works best with vertical bar charts, 2–4 bars."
- **Results area** (after upload), separated by hairline rules: data table → verdict stamp → original and redraw side by side.

On first load the upload box is essentially the whole app. "This is a tool, not a product site."

## Look and Feel
A newspaper data desk doing a fact-check, not a SaaS dashboard or a trading terminal.

- **Background:** warm off-white paper tone, like newsprint. Not stark white, not dark mode.
- **Typography:** a serif for the masthead, tagline, and headings. **Monospace for every number:** bar values, axis start, Lie Factor. Monospace reads as "precise data," measured rather than guessed.
- **Structure:** thin black hairline rules between sections. No rounded cards, soft shadows, or gradients.
- **Verdict stamp:** the app's visual signature. "MISLEADING" (red ink) or "LOOKS HONEST" (green ink), bold and slightly rotated, like an ink stamp on a document. It should look authoritative, not like a toast notification.
- **References:** PolitiFact's Truth-O-Meter crossed with the NYT Upshot's data-journalism style: serious, text-forward, confident verdicts.
- **Avoid:** generic AI-app styling, rounded-corner SaaS polish, gradients.

## Features and Behavior

### Upload
Source: `scope.md > The POC Boundary`.
- The user can drag an image onto the box or click to browse.
- When a file is accepted, the "Reading chart..." state appears immediately.
- [ ] Dropping or choosing an image starts reading with no extra button press.
- [ ] Clicking the box opens the system file picker.

### Extraction Table
Source: `scope.md > The POC Boundary`.
Two visually distinct groups:
- **Chart geometry** (pinned at the top, grouped together: "the things that control whether this chart is lying"):
  - **"Y-axis starts at: [n]"**, an editable number.
  - **"Axis direction: Ascending (normal) ⇅ Descending (inverted)"**, a two-option toggle rather than free text, since there are only two valid states. It defaults to what Unspin detected.
- **Chart data** (below): one row per bar with its label and an editable value, for example "My Fund: 14%" and "S&P 500: 11%".
- Extraction therefore produces three kinds of facts: bar labels and values, axis start, and axis direction.
- [ ] For the demo chart, the panel shows axis start 9, direction Ascending, My Fund 14, and S&P 500 11.
- [ ] Axis start, axis direction, and every bar value can be changed by the user.
- [ ] The geometry group (axis start and direction) looks visually distinct from the bar-value rows.

### Live Recalculation
- Any edit, whether to a bar value, the axis start, or the axis direction, recomputes the Lie Factor, the verdict text, the stamp, and the honest redraw immediately. There's no submit button and no page reload.
- [ ] Changing the demo chart's axis start from 9 to 0 visibly changes the Lie Factor to about 1, and the stamp to LOOKS HONEST.
- [ ] Changing a bar value updates the redraw's bar height.
- [ ] Flipping axis direction to Descending turns the stamp to MISLEADING with the "Axis inverted" verdict, and flipping it back restores the previous verdict.

### Lie Factor and Verdict
Source: `scope.md > The Unique Kernel`, `scope.md > Inverted Axis: Direction Is Separate from Size`.
- **Lie Factor** = (size of the effect shown in the graphic) / (size of the effect in the data).
  - Effect in the data: (larger value − smaller value) / smaller value.
  - Effect shown: the same calculation on the **drawn** bar heights, meaning each value minus the axis start.
  - It's computed on magnitudes. Direction is handled separately (see Inverted Axis).
- **3–4 bars: worst pair.** Compute the Lie Factor for every pair of bars and report the worst one, meaning the one furthest from 1. With a truncated axis, that's the highest. With exactly 2 bars there's only one pair, so no special case is needed.
- **Named pair:** directly under the stamp, show which two bars produced it. For example: "Biggest distortion: My Fund vs. S&P 500 — Lie Factor 5.5."
- **Threshold:** a Lie Factor from 0.9 to 1.1 counts as honest. Outside that range is misleading. The learner widened Tufte's 0.95–1.05 band to absorb extraction noise.
- The verdict names the trick and what it does, in plain language. For example: "⚠ Misleading — Lie Factor 5.5. Y-axis starts at 9 instead of 0."
- [ ] The demo chart (14 vs 11, axis start 9) shows MISLEADING with Lie Factor ≈ 5.5, and names the truncated axis.
- [ ] A chart with its axis starting at 0 shows LOOKS HONEST, Lie Factor ≈ 1.0.
- [ ] With 3–4 bars, the stamp shows the worst pair's Lie Factor and names that pair underneath.

### Truncated Y-Axis Detection
- Flagged when the y-axis starts above 0.
- [ ] The verdict text states where the axis actually starts ("starts at 9 instead of 0").

### Inverted Y-Axis Detection
Source: `scope.md > Inverted Axis: Direction Is Separate from Size`.
- Flagged when values increase downward, so the bar that looks bigger is actually the smaller value.
- An inverted chart is **always MISLEADING**, even if its Lie Factor magnitude falls within 0.9–1.1.
- Verdict example: "Axis inverted. The bar that looks taller (My Fund) is actually the smaller value. Lie Factor: 1.1 (magnitude is accurate, but direction is reversed)."
- [ ] An inverted-axis chart gets a MISLEADING stamp, and the verdict names which bar is visually reversed.
- [ ] A chart that is both truncated and inverted gets a verdict naming both tricks.

### Honest Redraw
Source: `scope.md > What "Working" Looks Like`.
- A clean bar chart of the same data, drawn upright with the axis starting at 0, in the app's newsprint style.
- Shown on the right. The original upload is on the left, exactly as uploaded.
- [ ] For the demo chart, the redraw shows two bars of nearly equal height (14 vs 11), next to the original where the fund's bar towers over the S&P's.
- [ ] For an honest chart, the redraw looks nearly identical to the original.

## States and Boundaries
- **First use:** only the masthead, context line, upload box, and expectation note are visible.
- **Reading:** "Reading chart..." is shown after upload.
- **Misleading result:** red MISLEADING stamp, Lie Factor, named trick(s), table, side by side.
- **Honest result:** the same layout with a green LOOKS HONEST stamp, for example "✓ Looks honest — Lie Factor 1.03. The chart's proportions match the data." The redraw still renders. "If Unspin only ever showed warnings, you couldn't tell if it was actually checking anything."
- **Recognized but unsupported chart:** if Unspin can tell what the chart is, it says so. For example: "This looks like a pie chart — Unspin currently only checks vertical bar charts." Shown where the table would be, with an option to upload a different image right there.
- **Unreadable image** (blurry, not a chart): "Couldn't read this as a bar chart. Try a clearer screenshot, or make sure it's a vertical bar chart with 2–4 bars." Shown in the same place, with the same re-upload option.
- **Persistence:** none. Refreshing the page clears everything. *(Assumption, consistent with scope's "no save/history/accounts.")*

## Product Decisions
- **Axis start is editable**, not just bar values, because it's one of the two inputs to the Lie Factor. A misread value nobody can fix means a wrong verdict.
- **Axis direction is editable too** (a toggle). A misread direction is arguably worse than a misread value, because it flips the verdict between honest and misleading entirely. It's grouped with axis start as "chart geometry," separate from "chart data," to reinforce what makes a chart lie.
- **Live recalculation**, no submit button. It also creates a second demo moment: you can watch the Lie Factor change as you fix a number.
- **Honest band 0.9–1.1**, wider than Tufte's 0.95–1.05, so extraction noise doesn't falsely flag an honest chart in front of a judge.
- **Honest charts get a full result** with a green stamp and a redraw, so warnings are believable.
- **Specific failure messages** when the chart type is recognizable. Generic fallback otherwise. Re-upload is offered in place.
- **Upfront expectation note** ("vertical bar charts, 2–4 bars"), so users aren't surprised by a failure.
- **Worst pair, not an average or fixed pair.** You can't assume the story is about the extremes or the leftmost and rightmost bars. A semantic "benchmark bar" would make extraction fragile ("it doesn't need to know what the bars mean, only what they're worth"). Averaging would dilute a severe localized distortion.
- **Size and direction are separate.** The Lie Factor measures magnitude. An inverted axis is flagged on its own and always makes the chart misleading.
- **Both tricks present → name both.** For example: "⚠ Misleading — Axis inverted, and starts at 9 instead of 0." Inversion is a separate always-flag rule and truncation drives the Lie Factor, so the two don't conflict.
- **Positive bar values only.** Negative values, like negative returns, would need a redraw with bars going both directions from zero. That's real added complexity the demo doesn't need.
- **Vertical bar charts only.** Horizontal bars are not in the POC.
- **Single page, no login, no nav.** "This is a tool, not a product site."

## What We're Building
- One-page app with masthead, context line, upload box (drag-and-drop or click), and expectation note.
- Extraction of three things from an uploaded vertical bar chart image: 2–4 bar labels and values, the axis start, and the axis direction.
- Editable panel: a chart-geometry group (axis start, axis direction toggle) above the chart-data rows, recalculating live.
- Lie Factor calculation and an honest/misleading verdict at the 0.9–1.1 threshold.
- Truncated-axis and inverted-axis detection, with plain-language verdict text.
- Stamp-style verdict in red or green.
- Honest redraw shown next to the original upload.
- Reading state, unsupported-chart message, unreadable-image message, and in-place re-upload.
- Newsprint data-desk visual style.
- **Works on any vertical bar chart with 2–4 bars, not just the demo chart.**

## Deferred From the POC
- **Saving, history, accounts:** nothing persists. Not needed to prove the kernel.
- **Shareable results or links:** they'd need storage and URLs. Out with persistence.
- **Batch upload:** one chart at a time proves the loop.
- **URL input or browser extension:** an uploaded image is enough for the demo.

## Possible Later Enhancements
- Line charts, pie charts, horizontal bar charts.
- Dual-axis detection; 3D or area distortion.
- Cherry-picked date-range detection.
- Multi-series charts, or more than 4 bars.

## Non-Goals
- **Not a fraud detector.** Unspin shows how a chart is drawn, not whether its numbers are true.
- **No AI paragraph explaining the chart.** The side-by-side pictures make the argument.
- **Not a product site:** no landing page, marketing copy, or navigation.
- **No negative bar values** in the POC (see Product Decisions).

## Open Questions
- **Can wait for `4-spec`:** accepted image formats.
