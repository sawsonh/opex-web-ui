---
parent: high-level-design
prefix: CHART
---

# Charting

## Context and Design Philosophy

Every reporting dimension shows metrics as interactive charts. Charting is the shared layer that turns a dimension's metric series into Plotly.js charts that look, behave, and label data quality the same way everywhere. It is the only module that imports Plotly.

Principles:

- **Dimensions describe, Charting draws.** A dimension builds a `ChartSpec` from its domain data — series, units, chart kind — and Charting renders it. Dimensions never build Plotly traces or layouts, so a convention changes in one place.
- **Never make data look better than it is.** A missing value is a gap, not a zero and not an interpolation. A partial period looks different from a complete one. Data freshness is always stated. (HLD tenet: accuracy over availability.)
- **Exact on demand.** Charts are dense; hover, the data table, and axis labels give the exact API value with its unit and period. (HLD tenet: analyst depth.)

## `ChartSpec`

```ts
type ChartSpec = {
  title: string;
  kind: 'line' | 'bar' | 'stackedBar' | 'scatter';
  unit: Unit;                       // 'count' | 'percent' | 'duration' | 'ratio'
  series: Series[];
  asOf: Instant | null;             // when the data was last aggregated, from the API
  stale: boolean;                   // the API flagged the data as stale
  bucket?: 'day' | 'week' | 'month' | 'sprint';  // time series only
};

type Series = {
  id: string;                       // stable; drives color assignment
  label: string;
  points: Point[];
};

type Point =
  | { x: CalendarDate | number; y: number; partial: boolean; ref?: PointRef }
  | { x: CalendarDate | number; y: null };          // no data for this bucket
```

`ChartSpec`, `Unit`, the formatters, and the color assignment are plain TypeScript in `shared/charting/model` with no Plotly or React import, so dimensions can build and unit-test specs without a DOM. The Plotly mapping and React components live in `shared/charting/plotly` and `shared/charting/ui`.

## Data Quality Presentation

| Condition (from the API) | How it is drawn | How it is labeled |
|---|---|---|
| Point with `y: null` | Lines break at the gap (`connectgaps: false`); no bar | Hover: "No data for {period}"; gaps count shown under the chart when any exist |
| Point with `partial: true` | Hollow marker, dashed segment into it; bar with hatch pattern | Hover: "Partial period — {period} is not complete"; legend note "Hollow / hatched = partial period" |
| Every point null, or no points | No chart; empty panel | "No data for this date range" and the date range |
| `stale: true` | Chart drawn normally | Warning bar above the chart: "Data may be out of date — last updated {asOf}" |
| Always | — | Footer: "Data as of {asOf}" in the user's local time zone, or "Data as of: unknown" when `asOf` is null |

Whether data is partial, missing, or stale is the API's determination. Charting never infers it — for example, it does not decide the current week is partial because today is mid-week — and when the API sends no such flags, Charting adds no warning of its own.

## Formatting

| Unit | Axis and hover format | Example |
|---|---|---|
| `count` | Integer with thousands separators | `1,204` |
| `percent` | Values are 0–100; one decimal place, `%` suffix | `83.4%` |
| `duration` | Values are seconds; shown in the largest unit giving a value ≥ 1, one decimal (`min`, `h`, `d`) | `3.2 h` |
| `ratio` | Two decimal places | `0.47` |

- Formatting uses the browser's locale for separators.
- Hover shows the series label, the formatted value, and the full period ("Week of 4 Aug 2026", "12 Aug 2026").
- Time-series x-values are **calendar dates naming a bucket**, displayed as dates without time-zone conversion — a daily bucket for 12 Aug is labeled 12 Aug for every viewer. Instants such as `asOf` are converted to the user's local time zone.

## Interaction

- Hover with a unified x tooltip for time series; per-point tooltip for scatter.
- Click a legend entry to hide or show a series; double-click to isolate it.
- Drag to zoom the x axis; double-click to reset. Zoom is a view-only change and does not alter the date range in the URL.
- The Plotly mode bar shows only "Download PNG" and "Reset axes".
- When a point carries a `ref` and the chart's `onPointSelect` is provided, clicking the point calls it with the `PointRef`. Dimensions use this for drill-down and for "Open RCA ticket" links.
- Each chart has a "Show table" toggle that renders the same series as an accessible HTML table — one row per x value, one column per series, empty cells for missing values marked "No data", partial values marked "(partial)". The chart container has an `aria-label` summarizing the title, date range, and series.

## Colors

- One categorical palette of 8 colors, distinguishable under common color-vision deficiencies. Colors are assigned by `Series.id` in order of first appearance on the page, so the same series keeps its color across the charts of a panel.
- Two semantic colors for direction-of-change indicators (better / worse), always paired with an arrow and text, never color alone. Which direction is better is supplied by the dimension.
- Partial and missing styles never rely on color alone.

## Plotly Integration

- Plotly is loaded as the `plotly.js-basic-dist-min` bundle (scatter, bar), in its own lazily loaded chunk, so routes without charts do not download it.
- `PlotlyChart` is a thin React component that calls `Plotly.react` on spec changes, `Plotly.purge` on unmount, and resizes with a `ResizeObserver`. It is the only component that touches Plotly's API.
- Charts render with `responsive: true`, the shared layout template (font, margins, grid, axis styles), and `displaylogo: false`.

## Panel States

`ChartPanel` wraps a chart with the states every dimension needs, so they read the same everywhere: loading (skeleton at the chart's size), error (from `ApiError`, with retry), empty, and loaded with the quality labels above.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Interface to dimensions | Plotly-free `ChartSpec` | Dimensions build Plotly traces directly | Keeps Plotly in one module, makes conventions (gaps, partials, formats) impossible to skip, and lets specs be unit-tested without a DOM. |
| React binding | Own thin `PlotlyChart` over `Plotly.react` | `react-plotly.js` | The wrapper is a few dozen lines; owning it avoids a lightly maintained dependency and controls bundle choice and cleanup. |
| Plotly bundle | `plotly.js-basic-dist-min`, lazy | Full `plotly.js-dist-min`; custom partial build | The basic bundle covers the chart kinds in use at about a third of the full size; a custom build adds build complexity for little extra saving. A new chart kind that needs another trace type revisits this. |
| Missing values | Gaps | Zero-fill; interpolate | Zero and interpolated values are values the API never returned. |
| Accessible alternative | "Show table" per chart | Only `aria-label` summaries | Analysts and screen-reader users both need exact values; the table serves both and doubles as a copy source. |
| Bucket date labels | Displayed as calendar dates, unconverted | Convert bucket starts to local time | A bucket names a day, not an instant; converting would shift labels across midnight for some viewers. |

## Open Questions & Future Decisions

### Deferred
1. How the API marks partial buckets, missing buckets, staleness, and `asOf` in metric responses.
2. A dark theme, if wanted.

## References

- `docs/high-level-design.md` — Presentation, not computation; Tenets
- `docs/intent/code-quality/code-quality-design.md` — first consumer
- `docs/intent/api-client/api-client-design.md` — `ApiError` for panel error states
