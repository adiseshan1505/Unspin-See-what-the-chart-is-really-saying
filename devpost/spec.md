---
doc: spec
status: approved
---

# Unspin — Technical Spec

## How This Works, In Plain Language
Unspin has two pieces that run on your laptop.

1. **A small Python server (FastAPI)** with one job. It receives the uploaded screenshot, sends it to Google's Gemini vision model along with a carefully written prompt and a JSON schema, and returns three facts about the chart: the **bar labels and values**, the **axis start**, and the **axis direction**. It exists only because the Gemini API key must stay secret. A key in browser code would be visible to anyone.
2. **A single web page (TypeScript + Vite)** that does everything else. It shows the upload box, puts the extracted facts in an editable panel, computes the Lie Factor (every pair of bars, keeping the worst), decides the verdict, stamps it, and draws the honest version with Chart.js next to your original image.

The server **reads** the chart. The browser **judges** it. Because all the math lives in the browser, editing a number in the panel recalculates the verdict and redraws the chart instantly, with no trip back to the server. That's what makes the live-recalculation demo moment feel live.

A third piece is for you, not users: an **evaluation script**. It generates 8–10 test charts with matplotlib where you know the true numbers, runs the real extraction on each, and prints how often Gemini got them right. That turns "it seemed to work" into "7 of 8 unlabeled charts extracted correctly."

Why this shape and nothing bigger: there's no database because nothing is saved, no login because there are no users to tell apart, and no deployment because the demo video is recorded on localhost.

## The Core Journey Through the System
PRD ref: `prd.md > The Core Journey`.

1. **Arrive.** You open `http://localhost:5173`. Vite serves `index.html`, and `main.ts` renders the masthead, context line, upload box, and expectation note.
2. **Upload.** You drop a PNG onto the box. `upload.ts` checks it's PNG/JPEG/WebP and under 10 MB, shows the original right away from a local object URL, switches to "Reading chart...", and calls `api.ts → POST /api/extract`.
3. **Read.** Vite's dev proxy forwards `/api/*` to FastAPI on port 8000. `main.py` validates the file and passes the bytes to `extract.py`, which sends the image, the prompt from `prompt.py`, and the JSON schema from `schema.py` to `gemini-3.8-flash`.
4. **Check.** Gemini returns JSON matching the schema. `extract.py` validates it with Pydantic and applies the POC rules: it must be a vertical bar chart with 2–4 bars and positive values. It returns either `{status: "ok", chart}` or `{status: "unsupported" | "unreadable", message}`.
5. **Judge.** The browser stores `chart` in its in-memory state. `lieFactor.ts` computes drawn heights, the Lie Factor for every pair, the worst pair, and the verdict.
6. **Show.** `panel.ts` renders the geometry group (axis start, direction toggle) and the data rows. `verdict.ts` renders the stamp, the Lie Factor, the trick text, and the "Biggest distortion" line. `compare.ts` shows the original on the left and draws the Chart.js redraw on the right.
7. **Correct.** Editing any field updates the state and re-runs step 5 and the verdict/redraw part of step 6, entirely in the browser.
8. **Again.** "Check another chart" clears the state and returns to the upload box. No reload, and nothing persists.

```
Browser (Vite, TS)                         Python (FastAPI)                Google
┌──────────────────────────┐  POST image   ┌────────────────────┐  image+  ┌────────────────┐
│ upload → panel → verdict │ ────────────→ │ /api/extract       │  prompt  │ gemini-3.8-    │
│   ↑ edits    ↓           │               │  extract.py        │ ───────→ │ flash          │
│ lieFactor.ts → redraw    │ ←──────────── │  validate (Pydantic)│ ←─────── │ (JSON schema)  │
└──────────────────────────┘  chart JSON   └────────────────────┘  JSON    └────────────────┘
   all math + rendering here                  only place the API key lives
```

## Stack
Learner's choices unless marked as derived.

- **Backend: Python 3.11+ with FastAPI.** It's the one part that needs a server, because of the hidden key. [FastAPI docs](https://fastapi.tiangolo.com/)
  - `uvicorn` to run it. [Uvicorn docs](https://www.uvicorn.org/)
  - `python-multipart` for file uploads, which FastAPI requires for `UploadFile` (derived). [Request files](https://fastapi.tiangolo.com/tutorial/request-files/)
  - `pydantic` (bundled with FastAPI) for the extraction schema and validation.
  - `python-dotenv` to load `GEMINI_API_KEY` from `backend/.env` (derived).
  - `google-genai`, the official Gemini Python SDK. [SDK docs](https://googleapis.github.io/python-genai/)
- **Vision model: Gemini, `gemini-3.8-flash`.** Learner's choice: good at structured image-to-JSON, with a usable free tier. Google's models page recommends 3.8 Flash for new projects. [Models](https://ai.google.dev/gemini-api/docs/models) · [Image understanding](https://ai.google.dev/gemini-api/docs/image-understanding) · [Structured output](https://ai.google.dev/gemini-api/docs/structured-output) · [Pricing](https://ai.google.dev/gemini-api/docs/pricing)
- **Frontend: plain TypeScript with Vite**, no framework. "It's a single page with a few interactive panels … a framework would add setup overhead without buying much." [Vite guide](https://vite.dev/guide/)
- **Redraw: Chart.js 4.x**, a lightweight bar chart that's easy to restyle. [Chart.js docs](https://www.chartjs.org/docs/latest/)
- **Math tests: Vitest** (derived). Small unit tests that pin the Lie Factor math to the PRD's numbers (5.5 and 1.0). It works with Vite with no extra config. [Vitest](https://vitest.dev/)
- **Eval charts: matplotlib** (learner's choice) for generating test charts with known values. [matplotlib bar](https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.bar.html)

**Verified on 2026-10-13 from Google's docs:** `gemini-3.8-flash` is the stable, recommended Flash model and has a free tier. Supported image types are PNG, JPEG, WebP, HEIC, and HEIF. Inline requests are limited to 20 MB. Structured output accepts a JSON Schema, which can come from Pydantic's `model_json_schema()`.

**Check in the first build step:** Google's current docs show a newer **Interactions API** (`client.interactions.create(..., response_format={"mime_type": "application/json", "schema": ...})`). Most other examples use the older `client.models.generate_content(..., config={"response_mime_type": "application/json", "response_json_schema": ...})`. One forum report says the docs' new structured-output sample errors on some versions. Make one real call with the installed SDK version and keep whichever path works. Only `extract.py` changes either way.

## Where It Runs and How Someone Tries It
- **Runtime:** local only, two processes. Python 3.11+ and Node 20+.
- **Secret:** `GEMINI_API_KEY` in `backend/.env`, which git ignores. `backend/.env.example` holds a placeholder and is committed. Get a key at [Google AI Studio](https://aistudio.google.com/apikey).
- **Start the backend:**
  ```
  cd backend
  python -m venv .venv && source .venv/bin/activate
  pip install -r requirements.txt
  uvicorn app.main:app --reload --port 8000
  ```
- **Start the frontend:**
  ```
  cd frontend
  npm install
  npm run dev        # opens http://localhost:5173
  ```
- **Run the evaluation:**
  ```
  cd backend
  python -m eval.make_charts   # writes eval/charts/*.png + *.json
  python -m eval.run_eval      # prints per-chart results + accuracy
  ```
- **Run the math tests:** `cd frontend && npm test`.
- **Demo recording:** record against `localhost:5173` using `backend/eval/charts/demo_fund_vs_sp500.png`, the scope's demo chart (14 vs 11, axis at 9), generated by `make_charts.py`. Then show an honest chart for the green stamp, and edit the axis start live.
- **Deployment:** none. Learner's choice: a live link "doesn't change how I'm judged," and it adds two deployments, key-exposure risk, and time. It's a stretch goal only if there's real buffer before the deadline. `6-ship` can revisit.

## Look and Feel
Carried from `prd.md > Look and Feel`. A newspaper data desk doing a fact-check. Concrete tokens are derived and live in `styles.css` as CSS variables:

- **Paper and ink:** background `--paper: #f6f1e7` (warm newsprint), text `--ink: #1b1b1b`, muted `--muted: #6b6457`, hairline rules `1px solid var(--ink)`. No cards, shadows, gradients, or rounded corners (`border-radius: 0` everywhere).
- **Type:** loaded from Google Fonts with system fallbacks.
  - Masthead and headings: **Playfair Display**, a serif with a masthead feel.
  - Body: **Source Serif 4**.
  - **Every number** (bar values, axis start, Lie Factor, chart ticks): **IBM Plex Mono**, falling back to `ui-monospace`.
- **Layout:** one centered column (max ~880px). Sections separated by hairline rules. Side-by-side compare in two columns, stacking on narrow screens.
- **Verdict stamp:** uppercase **MISLEADING** / **LOOKS HONEST**, heavy weight, wide letter-spacing, a double border in stamp ink, `transform: rotate(-4deg)`, slightly translucent ink. Red `--stamp-red: #b3261e`, green `--stamp-green: #2f6b3c`. It should look authoritative, not like a toast.
- **Geometry group:** set apart from the data rows, for example with a heavier top rule and small-caps label "CHART GEOMETRY", so it reads as "the things that control whether this chart is lying."
- **Upload box:** dashed 1.5px ink border on paper, serif prompt text. A subtle ink tint on drag-over.
- **Redraw styling:** Chart.js bars in solid ink, no gridline clutter (light hairlines only), ticks in Plex Mono, y-axis forced to start at 0. Caption: "Honest redraw — axis starts at 0."
- **Copy tone:** plain, confident, fact-check voice, from the PRD's examples. No AI-explanation paragraphs.
- **References:** PolitiFact Truth-O-Meter × NYT Upshot.

## Components

### Backend: Extract Endpoint (`app/main.py`)
`POST /api/extract`, multipart form field `file`.
- Rejects non-PNG/JPEG/WebP files or files over 10 MB with HTTP 400 and `{detail}`.
- Calls `extract_chart(bytes, mime_type)` and returns its result as JSON with HTTP 200, including "unsupported" and "unreadable", which are normal outcomes and not server errors.
- Returns HTTP 502 `{detail: "Couldn't reach the chart reader."}` if Gemini errors or times out (30 s).
- PRD ref: `prd.md > Upload`, `prd.md > States and Boundaries`.

### Backend: Gemini Extraction (`app/extract.py`)
- Builds one Gemini request: image bytes, mime type, the prompt from `prompt.py`, JSON output constrained by `GeminiChartReading` from `schema.py`, and model `gemini-3.8-flash`.
- Parses the reply with `GeminiChartReading.model_validate_json(...)`. A parse failure counts as unreadable.
- Applies the POC rules (see Data Model) and maps the result to the API response.
- The eval script imports `extract_chart` directly, so it tests exactly what the app uses.
- PRD ref: `prd.md > Extraction Table`, `prd.md > States and Boundaries`.

### Backend: Extraction Prompt (`app/prompt.py`)
The learner's focus area (see Decisions and Open Issues). It's kept in its own file so it can be read, edited, and evaluated on its own. The prompt instructs the model to:
- First identify the chart type (`vertical_bar`, `horizontal_bar`, `line`, `pie`, `other`, `not_a_chart`).
- Read the **y-axis tick labels**. `axis_start` is the value at the baseline the bars grow from. That isn't always the lowest tick: on an inverted axis, the baseline sits at the top.
- Set `axis_direction`: `ascending` if tick values increase upward, `descending` if they increase downward.
- For each bar, left to right: the label as printed, and the value. Use a printed data label if present; otherwise estimate from the bar's end against the gridlines and ticks.
- Report the unit if printed (for example `%`).
- Report `readable: false` rather than guessing when the image is too unclear.

### Backend: Schemas (`app/schema.py`)
Pydantic models for the Gemini reading and for the API response. See Data Model.

### Frontend: App Shell and State (`src/main.ts`)
Holds the single in-memory `AppState` and the `render()` flow. It connects upload → API → state → panel, verdict, and compare. Every edit calls `updateChart(patch)`, which recomputes the verdict and re-renders the verdict and the redraw.
PRD ref: `prd.md > Screens and Layout`, `prd.md > Live Recalculation`.

### Frontend: API Client (`src/api.ts`)
`extractChart(file: File): Promise<ExtractResult>`. It sends `FormData` to `/api/extract` and maps 400/502 and network errors to a displayable message.

### Frontend: Upload (`src/ui/upload.ts`)
The drag-and-drop box plus a hidden `<input type="file" accept="image/png,image/jpeg,image/webp">`. Dropping or choosing a file starts reading immediately. It checks type and size client-side, makes an object URL for the original, and shows "Reading chart...".
PRD ref: `prd.md > Upload`.

### Frontend: Editable Panel (`src/ui/panel.ts`)
- **Chart geometry group:** "Y-axis starts at" is a number input in Plex Mono. "Axis direction: Ascending (normal) ⇅ Descending (inverted)" is a two-option toggle.
- **Chart data rows:** each bar's label and a number input for its value.
- The `input` event, not `change`, triggers a live recompute on every keystroke. An empty or non-numeric field is held as invalid. The verdict shows "Check the panel" and the last valid redraw stays until the field is fixed.
- PRD ref: `prd.md > Extraction Table`, `prd.md > Live Recalculation`.

### Frontend: Lie Factor Engine (`src/lieFactor.ts`)
Pure functions with no DOM, unit-tested.
- `drawn(v) = v − axisStart`, the drawn bar height. For values above the baseline it's the same on either axis direction, so magnitude is computed the same way.
- `pairLieFactor(a, b)`:
  - Let `S` and `L` be the smaller and larger *values*.
  - `dataEffect = (L − S) / S`
  - `shownEffect = (drawn(L) − drawn(S)) / drawn(S)`
  - `LF = shownEffect / dataEffect`
- **Closed form (learner's derivation):** the `(L − S)` term cancels, so for any pair `LF = S / (S − axisStart)`. It depends only on the **smaller** value and the axis start, never the larger value.
  - Consequence: with one shared baseline, the worst pair (furthest from 1) **always includes the smallest bar**, so the chart's Lie Factor = `min / (min − axisStart)`.
- `worstPair(bars)`: all pairs (at most 6 for 4 bars), as the PRD specifies. Skip pairs with equal values, since there's no effect to exaggerate. Pick the one **furthest from 1**, measured as `max(LF, 1/LF)`.
  - **Tie-break:** pairs that share the smallest bar give identical Lie Factors, so the named pair is **smallest bar vs largest bar**, the most dramatic contrast in the original. *(Agent default; the learner can change it.)*
- **All values equal:** there's no difference to measure, so there's no Lie Factor and no stamp. A neutral note says "All bars have the same value — there's no difference for the axis to exaggerate." If the axis is also inverted, the inverted-axis warning still shows.
- `computeVerdict(chart)`:
  - `misleading = LF < 0.9 || LF > 1.1 || axisDirection === "descending"`
  - Tricks: `truncated` if `axisStart > 0`, `inverted` if descending. Both can be present.
  - Text: built from the PRD examples, naming both tricks when present, plus "Biggest distortion: A vs B — Lie Factor X."
- **LF display:** 2 decimals below 10 (`1.03`), 1 decimal at 10 and above. The demo shows **5.50**. *(Assumption: the PRD shows "5.5"; switch to 1 decimal at ≥ 2 if you prefer.)*
- **Degenerate inputs:** any bar with `value ≤ axisStart` makes `drawn ≤ 0`, so there's no valid Lie Factor. The verdict says "Check the panel: a bar's value is at or below the axis start." No stamp is shown.
- PRD ref: `prd.md > Lie Factor and Verdict`, `prd.md > Truncated Y-Axis Detection`, `prd.md > Inverted Y-Axis Detection`.

### Frontend: Verdict Stamp (`src/ui/verdict.ts`)
Renders the rotated stamp (red MISLEADING or green LOOKS HONEST), the Lie Factor in Plex Mono, the trick text, and the "Biggest distortion: A vs B" line (always shown by default; see Open Issues).
PRD ref: `prd.md > Lie Factor and Verdict`, `prd.md > States and Boundaries`.

### Frontend: Side-by-Side Compare (`src/ui/compare.ts`)
- Left: `<img>` of the original object URL, unchanged.
- Right: a Chart.js bar chart of the current `chart.bars`, with y from 0 and ascending, and newsprint styling.
- It keeps one Chart instance and updates its data with `chart.update()` on edits rather than rebuilding it, so live edits stay smooth.
- PRD ref: `prd.md > Honest Redraw`.

### Frontend: Messages (`src/ui/messages.ts`)
Renders the unsupported, unreadable, and network-error messages where the panel would be, with an in-place "Upload a different image" control.
PRD ref: `prd.md > States and Boundaries`.

### Evaluation Harness (`backend/eval/`)
- **`make_charts.py`** uses matplotlib to write each chart as a PNG plus a `.json` ground-truth file (`bars`, `axis_start`, `axis_direction`, `expected_status`). The set covers:
  1. The demo chart (labeled, axis at 9)
  2. The demo with no value labels
  3. An honest chart, labeled
  4. An honest chart, unlabeled
  5. A 3-bar truncated chart
  6. A 4-bar truncated chart
  7. An inverted-axis chart
  8. An alternate style (different colors and fonts)
  9. A pie chart (expects `unsupported`)
  10. Optionally, an inverted and truncated chart
- **`run_eval.py`** calls `extract_chart` on each chart and compares against ground truth:
  - **Bar values:** within ±2 units or ±5%, whichever is larger. This is the learner's "within a point or two."
  - **Axis start:** within ±0.25 units. It's tighter because the Lie Factor is most sensitive to axis-start error on truncated charts: reading 10 instead of 9 doubles the demo's LF from 5.5 to 11.
  - **Axis direction** and **status:** must match exactly.
  - **Verdict agreement:** recomputes the verdict from the extracted values with the same rules and checks it matches the verdict from the true values. This is the metric that matters most.
  - Prints a per-chart table plus totals, for example "values 7/8 · axis 8/8 · verdict 8/8."
- The verdict rules are mirrored in a small Python helper in `run_eval.py`. The test charts and the Vitest tests use the same demo numbers, so the two stay in agreement.
- PRD ref: `prd.md > What We're Building` ("works on any vertical bar chart with 2–4 bars, not just the demo").

## Data Model

### Gemini reading (`GeminiChartReading`, also the JSON schema sent to Gemini)
```
chart_type:      "vertical_bar" | "horizontal_bar" | "line" | "pie" | "other" | "not_a_chart"
readable:        boolean
axis_start:      number | null
axis_direction:  "ascending" | "descending" | null
unit:            string | null          # e.g. "%"
bars:            [{ label: string, value: number }]   # left to right; maxItems 12
```

### POC rules in `extract.py` → API response
| Condition | Response |
|---|---|
| `not_a_chart`, or `readable: false`, or schema parse fails | `{status: "unreadable", message: "Couldn't read this as a bar chart. Try a clearer screenshot, or make sure it's a vertical bar chart with 2–4 bars."}` |
| `pie` / `line` / `horizontal_bar` / `other` | `{status: "unsupported", detected_type, message: "This looks like a {pie chart} — Unspin currently only checks vertical bar charts."}` |
| vertical bar but bar count is not 2–4 | `{status: "unsupported", message: "This chart has {n} bars — Unspin currently checks 2–4."}` *(derived from the PRD's 2–4 boundary)* |
| any value ≤ 0, or `axis_start`/`axis_direction` missing | `unreadable` with the same message *(positive values only, per `prd.md > Product Decisions`)* |
| otherwise | `{status: "ok", chart: {axis_start, axis_direction, unit, bars}}` |

### Browser state (`AppState` in `main.ts`)
```
phase:        "empty" | "reading" | "result" | "message"
originalUrl:  string | null        # object URL of the uploaded file
chart:        { axisStart, axisDirection, unit, bars: [{label, value}] } | null
message:      string | null
```
- **Where it lives:** in memory in the browser tab only. The verdict is never stored. It's recomputed from `chart` on every change.
- **How it's updated:** the upload sets `originalUrl` and `phase: "reading"`. The API result sets `chart` or `message`. Panel edits patch `chart`.
- **When you leave and come back:** everything is gone. A refresh returns to the empty upload box, per `prd.md > States and Boundaries` (no persistence). "Check another chart" revokes the object URL and resets the state.
- **Server:** stores nothing. Images are not written to disk.

## File Structure
```
Unspin-See-what-the-chart-is-really-saying/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py          # FastAPI app: POST /api/extract, file checks, error mapping
│   │   ├── extract.py       # Gemini call + Pydantic validation + POC rules
│   │   ├── prompt.py        # the extraction prompt (learner's focus; iterate with eval)
│   │   └── schema.py        # GeminiChartReading + API response models
│   ├── eval/
│   │   ├── __init__.py
│   │   ├── make_charts.py   # matplotlib → charts/*.png + ground-truth *.json
│   │   ├── run_eval.py      # runs extract_chart on each, compares, prints accuracy
│   │   └── charts/          # generated test charts + truth files (incl. demo chart)
│   ├── requirements.txt     # fastapi, uvicorn, python-multipart, google-genai, python-dotenv, matplotlib
│   └── .env.example         # GEMINI_API_KEY=your-key-here
├── frontend/
│   ├── index.html           # page skeleton + Google Fonts links
│   ├── vite.config.ts       # dev proxy: /api → http://localhost:8000
│   ├── package.json         # vite, typescript, chart.js, vitest
│   ├── tsconfig.json
│   └── src/
│       ├── main.ts          # AppState, render flow, wiring
│       ├── api.ts           # POST /api/extract client
│       ├── types.ts         # Chart, Bar, ExtractResult, Verdict types
│       ├── lieFactor.ts     # pure math: drawn heights, pair LF, worst pair, verdict
│       ├── lieFactor.test.ts# Vitest: demo → 5.5, axis 0 → 1.0, LF unchanged when larger bar changes (12/16/20), inverted, 3–4 bars + tie-break, all-equal, degenerate
│       ├── styles.css       # newsprint tokens, stamp, rules, layout
│       └── ui/
│           ├── upload.ts    # drag-and-drop + file picker + reading state
│           ├── panel.ts     # chart geometry group + data rows (live inputs)
│           ├── verdict.ts   # stamp, LF, trick text, biggest-distortion line
│           ├── compare.ts   # original <img> + Chart.js honest redraw
│           └── messages.ts  # unsupported / unreadable / network messages + re-upload
├── devpost/                 # Devpost learning workspace (planning docs)
├── .gitignore               # learner profile, .env, .venv, node_modules, dist
├── README.md
└── LICENSE
```

## External Services and Dependencies

### Gemini API
- **Model:** `gemini-3.8-flash`.
- **Auth:** API key from `GEMINI_API_KEY`, which the `google-genai` client reads automatically. Used in `extract.py` only, never sent to the browser.
- **Call shape:** one request per upload, containing:
  - a text part: the prompt
  - an image part: the bytes plus `image/png|jpeg|webp`, inline (the 10 MB cap keeps us under the 20 MB inline limit)
  - a JSON response format constrained by `GeminiChartReading.model_json_schema()`
- **Response:** JSON text matching the schema. Validated with Pydantic. Structured output guarantees the *shape*, not that the values are *correct*. That's what the eval and the editable panel are for.
- **API surface:** Interactions API vs `generate_content`. Pick the one that works in the first build step (see Stack).
- **Cost:** free tier for 3.8 Flash input and output. Exact rate limits weren't checked; look them up in [AI Studio](https://aistudio.google.com/) if the eval hits limits. Running ~10 charts at a time is light.
- **Data use:** on the free tier, Google may use request content to improve its products. That's fine for synthetic test charts and public demo charts, but don't upload anything private. Mention this in the README.
- Docs: [Image understanding](https://ai.google.dev/gemini-api/docs/image-understanding) · [Structured output](https://ai.google.dev/gemini-api/docs/structured-output) · [Pricing](https://ai.google.dev/gemini-api/docs/pricing)

### Google Fonts
Playfair Display, Source Serif 4, and IBM Plex Mono, loaded via `<link>`. If they're offline, system serif and monospace fallbacks apply. [Google Fonts](https://fonts.google.com/)

## Important Failure Modes
- **Gemini misreads a value or the axis start** → this is the designed fallback, not a crash: the user fixes it in the panel and the verdict recomputes. The eval measures how often it happens.
- **Gemini is slow, errors, or hits a quota** → "Reading chart..." stays up for up to 30 s, then: "Couldn't reach the chart reader. Try again." with an in-place re-upload. The original image stays visible.
- **The model returns malformed or off-schema JSON** → treated as unreadable, with the generic message and re-upload.
- **Degenerate panel input** (a value at or below the axis start, or an empty or non-numeric field) → no stamp. A plain "Check the panel" note appears, and the last valid redraw stays until the input is fixed. Pairs with equal values are skipped in the worst-pair search.
- **All bars equal** → not an input error, since it can be a real chart. A neutral "no difference to exaggerate" note appears with no stamp (see Lie Factor Engine).

## What Was Simplified and Why
- **Local only, no deployment** instead of a hosted app. Learner's choice: the video and repo are what's judged, and deploying adds key-exposure risk and time. A deploy would need a host for each piece plus server-side key management.
- **Prompt-based extraction** instead of computer-vision pixel measurement of bar heights. One model call proves the kernel, and the editable panel covers errors. A CV pipeline would need edge detection, axis calibration, and OCR.
- **Synthetic test charts** instead of a scraped real-world set, giving exact ground truth (learner's reasoning). Real charts would need hand-labeling.
- **Chart.js redraw** instead of hand-drawn SVG. It's fast to build and restyled to fit the newsprint look.
- **In-memory state** instead of any storage. The PRD has no persistence.
- **Server holds nothing.** Images go straight from memory to Gemini and are discarded.

## Decisions and Open Issues

### Learner decisions
- **FastAPI backend with exactly one job** (vision call plus JSON). It exists because the API key must stay hidden.
- **All downstream logic in browser TypeScript**, so edits recompute instantly with no round trip. This is what makes live recalculation feel live.
- **Plain TypeScript + Vite, no React.** It's a single page, and a framework would add setup without benefit.
- **Chart.js** for the redraw.
- **Gemini** for extraction: structured image-to-JSON and a free tier.
- **Local plus recording; no deployment.** Deploying is a stretch goal only with real buffer.
- **Synthetic matplotlib eval set (8–10 charts) with ground-truth JSON and tolerances**, giving a real accuracy number. Imperfect extraction is covered by the editable panel by design.

### Derived implementation details (agent, following from the above)
- Model `gemini-3.8-flash` (Google's recommendation for new projects).
- Vite dev proxy instead of CORS configuration.
- 10 MB upload cap; PNG, JPEG, and WebP accepted. This closes `prd.md > Open Questions` on image formats.
- Separate, tighter axis-start tolerance (±0.25) in the eval, plus a verdict-agreement metric.
- "Worst" means furthest from 1, measured as `max(LF, 1/LF)`.
- More than 4 bars → a specific unsupported message.
- Vitest unit tests for the math.
- Concrete fonts and colors in Look and Feel.

### The learning thread: prompting a vision model for reliable structured JSON
- **Learner's question:** how to prompt a vision model to reliably return bar values, axis start, and axis direction as clean JSON, understanding it rather than copy-pasting a working call.
- **What clarifies it so far:** reliability has three separate layers, each with its own home in this spec:
  1. **Shape:** the JSON schema passed to Gemini (`schema.py`) constrains the output to valid JSON with the right fields and enum values. That's a guarantee from the API, not from the prompt.
  2. **Meaning:** the prompt (`prompt.py`) tells the model *how to read* a chart: classify first, use tick labels to find the baseline, decide direction from tick order, and prefer printed labels over estimating from gridlines. Most real accuracy gains come from here.
  3. **Trust:** Pydantic validation plus the POC rules (`extract.py`) reject readings that are well-formed but unusable.
- **Investigation during the build:** write the first prompt yourself in `prompt.py`, run `run_eval.py`, and change **one thing at a time**: an instruction, the order of fields, or the thinking level. Re-run after each change. **Evidence:** a before/after accuracy comparison (for example "unlabeled values 4/8 → 7/8 after adding the gridline instruction"), noted in `learner-profile.md > Learning Moments`.

### Learner finding: the Lie Factor depends only on the smaller value
`LF = S / (S − a)`, derived by hand and checked with L = 12, 16, and 20 (S = 11, a = 9 → 5.5 each time). Consequences:
- **Extraction risk is asymmetric.** Misreading the larger bar doesn't change the verdict. Misreading the **smallest bar or the axis start** does. This is why **verdict agreement** is the eval's headline metric; per-field tolerances are diagnostics.
- **The worst pair always includes the smallest bar** (one shared baseline), which needs the tie-break above. The PRD's worry about distortion "hiding between the 2nd and 3rd bar" can't occur with a uniform truncation, but the all-pairs rule is still correct and stays as specified.
- **The honest band protects against axis-start misreads specifically.** An axis read as exactly 0 gives LF = 1.0 however badly the values are misread. A false "misleading" needs a misread axis start of at least about S/11.
- `prd.md > Inverted Y-Axis Detection`'s example "Lie Factor: 1.1" is illustrative. An inverted chart with its baseline at 0 gives exactly 1.0, and the build shows the computed value.

### Open issues
- **Verify early (first build step):** whether the Interactions API or `generate_content` works for image + JSON schema with the installed `google-genai` version. Also confirm `gemini-3.8-flash` is reachable with your key.
- **Small choice, can decide during the build:** show the "Biggest distortion: A vs B" line for 2-bar charts too, or only for 3–4 bars? The PRD's example shows it for the 2-bar demo, so the default is **always show**.
- **Small choice:** LF decimal display (see the Lie Factor Engine assumption).
