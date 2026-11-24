---
parent: high-level-design
prefix: CQ
---

# Code Quality

## Context and Design Philosophy

Code Quality is the reporting dimension for static-analysis results collected from SonarQube. It shows a tenant's Bugs, Code Smells, Vulnerabilities, Duplication Density, and Code Coverage over the selected date range, at the tenant level and broken down by the SonarQube projects the tenant's configuration collects.

It is also the reference shape for every reporting dimension: the same four layers, the same split between domain rules and presentation, and the same use of Charting. Other dimensions follow it and differ only in their metrics.

Principles:

- **Show the API's numbers.** Tenant-level values, per-project values, and change over the range all come from the API. The UI does not sum projects, average coverage, or subtract start from end. (HLD tenet: the API owns the numbers.)
- **Name the origin.** Code Coverage here is SonarQube's measurement and is labeled "Code Coverage (SonarQube)", distinct from the Test Coverage dimension's GitHub-sourced coverage.

## Metrics

| Metric id | Label | Unit | Better when | Meaning (defined upstream) |
|---|---|---|---|---|
| `cq.bugs` | Bugs | count | lower | Open bug issues at the end of each bucket |
| `cq.code-smells` | Code Smells | count | lower | Open code-smell issues at the end of each bucket |
| `cq.vulnerabilities` | Vulnerabilities | count | lower | Open vulnerability issues at the end of each bucket |
| `cq.duplication-density` | Duplication Density | percent | lower | Duplicated lines as a share of lines |
| `cq.code-coverage` | Code Coverage (SonarQube) | percent | higher | Covered lines/conditions as SonarQube reports |

Definitions belong to `opex-collector-worker` and `opex-web-api`; the table records how the UI labels and presents each metric. When the API serves a severity breakdown for Bugs or Vulnerabilities (blocker, critical, major, minor, info), it is shown as a stacked bar chart; otherwise the metric shows a single series.

## Page Layout

```
Code Quality · acme · Last 30 days (24 Jul – 22 Aug 2026)        Granularity: [Day|Week|Month]
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Bugs     │ Smells   │ Vulns    │ Dup.     │ Coverage │   ← summary tiles
│ 42  ↓ 6  │ 310 ↑ 12 │ 3   — 0  │ 4.1% ↓0.3│ 78.2% ↑1.1│
└──────────┴──────────┴──────────┴──────────┴──────────┘
┌─ Bugs ─────────────────────────┐ ┌─ Vulnerabilities ─────────────┐
│ time series (tenant level)     │ │ time series (tenant level)     │
└────────────────────────────────┘ └────────────────────────────────┘
┌─ Code Smells ──────────────────┐ ┌─ Duplication Density ──────────┐
└────────────────────────────────┘ └────────────────────────────────┘
┌─ Code Coverage (SonarQube) ────┐
└────────────────────────────────┘
┌─ By project ────────────────────────────────────────────────────┐
│ Project       Bugs  Smells  Vulns  Dup.   Coverage   (sortable)  │
└─────────────────────────────────────────────────────────────────┘
```

The page is built from Charting's dimension page kit: `MetricTiles` (all five metrics are `snapshot`), one `ChartPanel` per metric at tenant level, `SourceItemTable` of SonarQube projects with comparison, `GranularityControl`, `NoSourceItems`, and `OpenRcaTicket`. Code Quality supplies:

- its metric definitions (the table above, as `DimensionMetric`s);
- the source-item table's default sort: Bugs descending;
- the severity breakdown panels for Bugs and Vulnerabilities when the API provides them;
- the `MetricRef` for "Open RCA ticket": metric id, tenant, date range, and optionally the project and point.

## Layers

| Layer | Contents |
|---|---|
| `domain/` | `CodeQualityMetricId`, the metric table above (labels, units, direction), `CodeQualityOverview` (tiles, tenant series, per-project rows), the rule turning a change and direction into better/worse/unchanged. No React, Plotly, or HTTP. |
| `application/` | `CodeQualityRepository` port; `loadOverview(tenant, range, granularity)`; `toChartSpecs(overview, selectedProjects)` building Charting specs from the domain model. |
| `infrastructure/` | Adapter implementing the port with the generated client; maps API DTOs to domain types, including null values, partial flags, `asOf`, and staleness. Unknown metric ids from the API are ignored, not shown. |
| `ui/` | Page composed from the dimension page kit; TanStack Query hooks keyed by tenant, range, granularity, and comparison. |

- The area descriptor: `id: 'code-quality'`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, `usesDateRange: true`.
- The Code Quality read routes are added to the API Client's `DEMO_READ_ROUTES`.

## Data States

- Loading, error, empty, partial, and stale data are presented through Charting's `ChartPanel` and the quality rules there.
- A tenant whose configuration collects no SonarQube projects shows `NoSourceItems` ("No SonarQube projects are configured for this tenant") in place of the panels.
- A project that is in the API's results but has no data in the range is listed with "No data" rather than omitted.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Tenant totals and change | From the API | Sum projects / subtract in the browser | Coverage and duplication cannot be summed, and even counts would define the metric in two places. |
| Severity breakdown | Stacked bars when the API provides it | Always a single series | Severity drives triage of bugs and vulnerabilities, but only the API can say how issues split. |
| Coverage naming | "Code Coverage (SonarQube)" | "Code Coverage" | Two dimensions report coverage from different origins; the label must say which. |
| Unknown metrics in responses | Ignored | Rendered generically | A metric needs a label, unit, and direction defined here before it can be presented honestly. |

## Open Questions & Future Decisions

### Deferred
1. The Code Quality endpoints: whether one overview call returns tiles, series, and project rows, or several calls do; and whether the API provides change-over-range and severity breakdowns.
2. Whether SonarQube's new-code (leak period) values should be shown alongside overall values.

## References

- `docs/high-level-design.md` — Domain: reporting dimensions and metrics; Module structure
- `docs/intent/charting/charting-design.md` — `ChartSpec`, `ChartPanel`, data quality presentation
- `docs/intent/app-shell/app-shell-design.md` — area descriptor, date range
- `docs/intent/api-client/api-client-design.md` — `DEMO_READ_ROUTES`
- `docs/intent/auth-session/auth-session-design.md` — `can()`
