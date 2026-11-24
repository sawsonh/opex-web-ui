---
parent: high-level-design
prefix: WT
---

# Work Throughput

## Context and Design Philosophy

Work Throughput is the reporting dimension for ticket flow collected from Jira. It shows how many tickets a tenant closes and how long work stays in progress, at the tenant level and by Jira project, over the selected date range.

It follows the reference shape set by Code Quality (`docs/intent/code-quality/code-quality-design.md`): four layers, pages built from Charting's dimension page kit (tiles, one chart panel per metric, source-item table with comparison, granularity control, "Open RCA ticket"). This document records what is specific to Work Throughput.

## Metrics

| Metric id | Label | Unit | Better when | Meaning (defined upstream) |
|---|---|---|---|---|
| `wt.tickets-closed` | Tickets Closed | count | — (neutral) | Tickets moved to a done status within each bucket |
| `wt.wip-cycle-time` | WIP Cycle Time | duration | lower | First in-progress status → done, for tickets closed in the bucket |

- **Flow, not snapshot.** Tickets Closed counts events within each bucket and is drawn as bars; the summary tile shows the API's total for the range, not the last bucket.
- **Neutral direction for Tickets Closed.** Closing more tickets is not better on its own (ticket size varies), so its change indicator shows direction without "better"/"worse".
- **Issue type breakdown.** When the API breaks Tickets Closed down by issue type (story, bug, task, …), the panel is a stacked bar chart.
- WIP Cycle Time is labeled with the statistic the API returns, with the bucket's ticket count in the hover, as in Development Velocity.

## Breakdown

Breakdown items are Jira projects, labeled by project key and name. Default sort: Tickets Closed descending.

## Layers and Registration

| Layer | Specific contents |
|---|---|
| `domain/` | `WorkThroughputMetricId`, the metric table, each metric's tile aggregation (`flow` for Tickets Closed, `range` for WIP Cycle Time) |
| `application/` | `WorkThroughputRepository` port; `loadOverview`; `toChartSpecs` |
| `infrastructure/` | Adapter over the generated client |
| `ui/` | Page composed from Charting's dimension page kit |

Area descriptor: `id: 'work-throughput'`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, `usesDateRange: true`. Its read routes are added to `DEMO_READ_ROUTES`.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Tickets Closed direction | Neutral | Higher is better | Raw counts reward splitting tickets; a "better" arrow would encourage that. |
| Tickets Closed tile | API's total for the range | Last bucket's value | A flow metric's meaningful summary is the period total; using the last bucket would vary with granularity. |

## Open Questions & Future Decisions

### Deferred
1. Which Jira statuses the collector treats as in progress and done, and whether that is configurable per tenant.

## References

- `docs/intent/code-quality/code-quality-design.md` — reference shape for dimensions
- `docs/intent/development-velocity/development-velocity-design.md` — statistic labeling and sample size
- `docs/intent/charting/charting-design.md`
