# Charting — EARS Specs

Design: `charting-design.md`

## Model

- [ ] **CHART-MODEL-001**: The charting model (`ChartSpec`, `Unit`, formatters, color assignment) shall have no dependency on Plotly, React, or the DOM.
- [ ] **CHART-MODEL-002**: The charting module shall be the only module that imports Plotly, and `PlotlyChart` the only component that calls Plotly's API.
- [ ] **CHART-MODEL-003**: When a `pie` `ChartSpec` has more or fewer than one series, the charting module shall reject it as invalid.

## Data quality

- [ ] **CHART-QUAL-001**: When a time-series point has `y: null`, the chart shall leave a gap at that period (lines broken, no bar), and its hover shall read "No data for {period}".
- [ ] **CHART-QUAL-002**: When a chart has one or more `y: null` points, the chart panel shall state the number of periods without data under the chart.
- [ ] **CHART-QUAL-003**: The charting module shall never draw a missing value as zero and never interpolate across a missing value.
- [ ] **CHART-QUAL-004**: When a point has `partial: true`, the chart shall draw it with a hollow marker and a dashed segment into it (line charts) or a hatched bar (bar charts), its hover shall read "Partial period — {period} is not complete", and the legend shall note "Hollow / hatched = partial period".
- [ ] **CHART-QUAL-005**: When a `ChartSpec` has no points or only `y: null` points, the chart panel shall show no chart and the message "No data for this date range" with the date range.
- [ ] **CHART-QUAL-006**: When a `ChartSpec` has `stale: true`, the chart panel shall show a warning bar above the chart reading "Data may be out of date — last updated {asOf}".
- [ ] **CHART-QUAL-007**: When a `ChartSpec` has an `asOf`, the chart panel shall show a footer "Data as of {asOf}" in the browser's time zone; when `asOf` is null, it shall show no footer.
- [ ] **CHART-QUAL-008**: The charting module shall mark data as partial, missing, or stale only when the `ChartSpec` says so, and shall add no data-quality warning of its own when the API supplies no such flags.

## Formatting

- [ ] **CHART-FMT-001**: The charting formatters shall format `count` values as integers with thousands separators, `percent` values (0–100) with one decimal place and a `%` suffix, `ratio` values with two decimal places, and `duration` values (seconds) in the largest of minutes, hours, or days giving a value of at least 1, with one decimal place.
- [ ] **CHART-FMT-002**: The charting formatters shall use the browser's locale for number separators.
- [ ] **CHART-FMT-003**: A chart's hover shall show the series label, the formatted value, and the full period name.
- [ ] **CHART-FMT-004**: The charting module shall display time-series x-values (bucket dates) as calendar dates without time-zone conversion, and instants such as `asOf` in the browser's time zone.
- [ ] **CHART-FMT-005**: When a point has `sampleSize`, its hover shall show "based on {count} {noun}" with the noun pluralized to match the count.
- [ ] **CHART-FMT-006**: When a point has `details`, its hover shall show each detail as "{label}: {value}" after the value, in order.
- [ ] **CHART-FMT-007**: When a `pie` slice has `share`, its hover and label shall show that share as the percentage; Plotly's computed percentages shall be turned off, and a slice without `share` shall show its count only.

## Interaction

- [ ] **CHART-INT-001**: Time-series charts shall show one unified hover per x value; scatter charts shall show a hover per point.
- [ ] **CHART-INT-002**: Clicking a legend entry shall hide or show its series; double-clicking shall isolate it.
- [ ] **CHART-INT-003**: Dragging on a time-series chart shall zoom its x axis and double-clicking shall reset it, without changing the date range in the URL.
- [ ] **CHART-INT-004**: The chart's mode bar shall offer only "Download PNG" and "Reset axes", and shall not show the Plotly logo.
- [ ] **CHART-INT-005**: When a point with a `ref` is clicked and the chart has an `onPointSelect` handler, the chart shall call the handler with the point's `PointRef`.
- [ ] **CHART-INT-006**: When a point with a `link` is clicked, the chart shall offer the link as a plain link opening in a new tab with `rel="noopener noreferrer"`, and shall make no request to it.
- [ ] **CHART-INT-007**: Each chart shall have a "Show table" toggle that shows the same data as an HTML table with one row per x value and one column per series, marking missing values "No data" and partial values "(partial)".
- [ ] **CHART-INT-008**: Each chart container shall have an `aria-label` summarizing the chart's title, date range, and series.

## Colors

- [ ] **CHART-COLOR-001**: The charting module shall assign series colors from an 8-color palette distinguishable under common color-vision deficiencies, by `Series.id` in order of first appearance on the page, so a series keeps its color across the page's charts.
- [ ] **CHART-COLOR-002**: When a metric's direction is `higher` or `lower`, its change indicator shall show an arrow, the word "better" or "worse", and the matching semantic color.
- [ ] **CHART-COLOR-003**: When a metric's direction is `neutral`, its change indicator shall show an arrow and the word "up" or "down" in a neutral color, without "better" or "worse".
- [ ] **CHART-COLOR-004**: The charting module shall never convey partial data, missing data, or direction of change by color alone.

## Plotly integration

- [ ] **CHART-PLOTLY-001**: The charting module shall load Plotly as the `plotly.js-basic-dist-min` bundle in a lazily loaded chunk that routes without charts do not request.
- [ ] **CHART-PLOTLY-002**: `PlotlyChart` shall call `Plotly.react` when its spec changes, `Plotly.purge` when it unmounts, and resize the chart when its container's size changes.

## Panel states

- [ ] **CHART-PANEL-001**: While a chart's data is loading, `ChartPanel` shall show a skeleton at the chart's size.
- [ ] **CHART-PANEL-002**: If a chart's data fails to load, then `ChartPanel` shall show the error for the `ApiError` kind with a retry.

## Dimension page kit

- [ ] **CHART-KIT-001**: `DimensionPage` shall show, in order, a title line (dimension · tenant · resolved date range), the granularity control, the metric tiles, the chart panels, and the source-item table.
- [ ] **CHART-KIT-002**: For a metric whose aggregation is `snapshot`, `MetricTiles` shall show the API's latest value in the range; for `flow`, the API's total for the range; for `range`, the API's value computed over the whole range.
- [ ] **CHART-KIT-003**: `MetricTiles` shall show the API's change over the range with the metric's direction indicator, show "—" for a value the API did not provide, and show no indicator when the API provides no change.
- [ ] **CHART-KIT-004**: `SourceItemTable` shall show one row per source item with the API's values per metric, sortable by any column, initially sorted by the dimension's default sort, with "—" for missing values and "No data" for an item with no data in the range.
- [ ] **CHART-KIT-005**: When the user checks a row in `SourceItemTable`, every time-series panel on the page shall add that source item's series beside the tenant-level series; unchecking shall remove it.
- [ ] **CHART-KIT-006**: While 5 source items are checked for comparison, `SourceItemTable` shall disable the unchecked rows and state why.
- [ ] **CHART-KIT-007**: The comparison selection shall be read from and written to the query string as `compare=`, and cleared when the user switches tenant.
- [ ] **CHART-KIT-008**: `GranularityControl` shall offer only the granularities (day, week, month, sprint) the API supports for the date range, read and write the choice as `granularity=` in the query string, and fall back to the API's default when the current choice is unsupported.
- [ ] **CHART-KIT-009**: `NoSourceItems` shall show "No {source items} are configured for this tenant" and, when `can(user, 'editTenantConfiguration', tenant)`, a link to the tenant's Settings.
- [ ] **CHART-KIT-010**: `OpenRcaTicket` shall appear in each chart panel's menu and on point click only when `can(user, 'writeImprovementTickets', tenant)`, and shall navigate to the Improvement area's new-ticket route with a `MetricRef`.
