---
parent: high-level-design
prefix: TC
---

# Test Coverage

## Context and Design Philosophy

Test Coverage is the reporting dimension for test-coverage trends collected from GitHub — coverage reports produced by the tenant's repositories' CI and gathered by `opex-collector-worker`. It shows the tenant's coverage over the selected date range, at the tenant level and broken down by the GitHub repositories the tenant's configuration collects.

It follows the reference shape set by Code Quality (`docs/intent/code-quality/code-quality-design.md`): the same four layers, pages built from Charting's dimension page kit (tiles, one chart panel per metric, source-item table with comparison, granularity control, "Open RCA ticket"). This document records only what is specific to Test Coverage.

All values — tenant level, per repository, and change over the range — come from the API. The UI does not combine repositories' coverage.

## Metrics

| Metric id | Label | Unit | Better when | Meaning (defined upstream) |
|---|---|---|---|---|
| `tc.line-coverage` | Line Coverage (GitHub) | percent | higher | Covered lines as a share of coverable lines, from the repositories' coverage reports |
| `tc.branch-coverage` | Branch Coverage (GitHub) | percent | higher | Covered branches as a share of branches; shown only when the API provides it |

- Labels carry "(GitHub)" to distinguish them from Code Quality's "Code Coverage (SonarQube)".
- A repository whose CI publishes no coverage report appears in the breakdown table with "No coverage reported" rather than 0%.

## Breakdown

- Breakdown items are GitHub repositories, labeled `owner/name`.
- Default sort: Line Coverage ascending, so the least-covered repositories come first.

## Layers and Registration

| Layer | Specific contents |
|---|---|
| `domain/` | `TestCoverageMetricId`, the metric table above |
| `application/` | `TestCoverageRepository` port; `loadOverview(tenant, range, granularity)`; `toChartSpecs` |
| `infrastructure/` | Adapter over the generated client; maps "no report" distinctly from a 0% value |
| `ui/` | Page composed from Charting's dimension page kit |

Area descriptor: `id: 'test-coverage'`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, `usesDateRange: true`. Its read routes are added to `DEMO_READ_ROUTES`.

A tenant collecting no GitHub repositories shows "No GitHub repositories are configured for this tenant", with a link to tenant configuration for users with `editTenantConfiguration`.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| No report vs. 0% | Distinct "No coverage reported" state | Treat missing as 0% | A repository without a coverage report is unmeasured, not untested; showing 0% would misstate it. |
| Default sort | Lowest coverage first | Alphabetical | Analysts open this view to find where coverage is weakest. |

## Open Questions & Future Decisions

### Deferred
1. Which coverage measures the collector extracts (line, branch, others) and from which report formats.
2. Whether per-pull-request coverage change is collected and should be shown.

## References

- `docs/intent/code-quality/code-quality-design.md` — reference shape for dimensions
- `docs/intent/charting/charting-design.md`
- `docs/high-level-design.md` — Domain: reporting dimensions and metrics
