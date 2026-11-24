---
parent: high-level-design
prefix: DV
---

# Development Velocity

## Context and Design Philosophy

Development Velocity is the reporting dimension for pull-request flow collected from GitHub. It shows how long pull requests take to be engaged with and merged, how complex they are, and how review attention relates to complexity, at the tenant level and by repository, over the selected date range.

It follows the reference shape set by Code Quality (`docs/intent/code-quality/code-quality-design.md`): four layers, pages built from Charting's dimension page kit (tiles, one chart panel per metric, source-item table with comparison, granularity control, "Open RCA ticket"). This document records what is specific to Development Velocity: duration statistics and the per-pull-request scatter view.

## Metrics

| Metric id | Label | Unit | Better when | Meaning (defined upstream) |
|---|---|---|---|---|
| `dv.pr-cycle-time` | PR Cycle Time | duration | lower | Ready for Review → Merge |
| `dv.pr-time-to-first-engagement` | Time to First Engagement | duration | lower | Ready for Review → first review or comment by someone other than the author |
| `dv.pr-first-engagement-to-merge` | First Engagement → Merge | duration | lower | First engagement → merge |
| `dv.pr-complexity` | PR Complexity | count | lower | The collector's complexity score per pull request |
| `dv.pr-complexity-comment-ratio` | Complexity vs. Comments | ratio | — (neutral) | Review comments relative to complexity |

- **Statistics.** For each bucket the API returns a statistic per metric (for example the median, and the 90th percentile when available). Each series is labeled with the statistic the API names ("PR Cycle Time — median"). The UI never computes a statistic from per-PR data.
- **Neutral direction.** Whether a given ratio is healthy depends on how it combines with complexity and cycle time — many comments on a complex pull request can be healthy engagement that lengthens cycle time, while few comments and a short cycle time can be a concern. Interpreting that is `opex-insights-workflow`'s job, so the ratio's change indicator shows the direction without "better"/"worse".
- **Sample size.** Each bucket's pull-request count, when the API provides it, appears in the hover ("based on 14 PRs"), so a statistic over very few pull requests is visibly thin.

## Per-Pull-Request Scatter

A panel plots one point per merged pull request in the range: complexity on x, review comments on y. Hovering shows the repository, PR number and title, complexity, comments, and cycle time. When the API provides the pull request's GitHub URL, clicking a point offers "Open in GitHub" (a plain link in a new tab; the UI makes no call to GitHub). The scatter respects the breakdown selection: with repositories selected, only their pull requests are plotted, colored by repository.

When the API returns more pull requests than it will list per page, the panel states how many are plotted out of the total.

## Breakdown

Breakdown items are GitHub repositories (`owner/name`). Default sort: PR Cycle Time (median) descending.

## Layers and Registration

| Layer | Specific contents |
|---|---|
| `domain/` | `DevelopmentVelocityMetricId`, the metric table, statistic labels, the `neutral` direction |
| `application/` | `DevelopmentVelocityRepository` port; `loadOverview`; `loadPullRequests(tenant, range, repositories)`; `toChartSpecs` |
| `infrastructure/` | Adapter over the generated client |
| `ui/` | Page composed from Charting's dimension page kit plus the scatter panel |

Area descriptor: `id: 'development-velocity'`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, `usesDateRange: true`. Its read routes, including the per-PR list, are added to `DEMO_READ_ROUTES`.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Duration statistic | Whatever the API returns, labeled | UI computes median/p90 from per-PR data | One definition, upstream; the UI only labels it. |
| Comment ratio direction | Neutral | Lower is better; higher is better | The ratio is healthy or worrying only in combination with complexity and cycle time; that judgment belongs to `opex-insights-workflow`, not to a UI arrow. |
| Complexity vs. comments view | Per-PR scatter alongside the ratio series | Ratio series only | The scatter shows outliers (large PRs with no review) that an aggregate ratio hides. |
| PR links | Plain link to GitHub when the API supplies the URL | Embedding PR details from GitHub | The UI never calls source systems. |

## Open Questions & Future Decisions

### Deferred
1. The complexity score's definition and range, and which statistics the API returns per duration metric.
2. The per-PR list endpoint's paging limit.

## References

- `docs/intent/code-quality/code-quality-design.md` — reference shape for dimensions
- `docs/intent/charting/charting-design.md`
- `docs/high-level-design.md` — Domain: reporting dimensions and metrics
