---
parent: high-level-design
prefix: SH
---

# Support Health

## Context and Design Philosophy

Support Health is the reporting dimension for incident response collected from PagerDuty. It shows how many incidents a tenant has, where they come from, how many are documented with notes, and how quickly they are acknowledged and resolved, at the tenant level and by PagerDuty service, over the selected date range.

It follows the reference shape set by Code Quality (`docs/intent/code-quality/code-quality-design.md`): four layers, pages built from Charting's dimension page kit (tiles, one chart panel per metric, source-item table with comparison, granularity control, "Open RCA ticket"). This document records what is specific to Support Health.

## Metrics

| Metric id | Label | Unit | Better when | Meaning (defined upstream) |
|---|---|---|---|---|
| `sh.incident-count` | Incident Count | count | lower | Incidents triggered within each bucket |
| `sh.incidents-with-notes` | Incidents with Notes | count | — (neutral) | Incidents triggered within each bucket that have at least one note, beside those without |
| `sh.mtta` | MTTA (Mean Time to Acknowledge) | duration | lower | Mean of trigger → first acknowledgement, for incidents triggered in the bucket |
| `sh.mttr` | MTTR (Mean Time to Resolve) | duration | lower | Mean of trigger → resolve, for incidents resolved in the bucket |

- **Incident Count and Incidents with Notes are flow metrics**, drawn as bars; their tiles show the API's totals for the range. Incidents with Notes is a stacked bar per bucket — with notes and without — so the share documented is visible without the UI computing it.
- **Incident count beside the means.** MTTA and MTTR hovers also show the bucket's incident count ("mean of 3 incidents"). A mean over a handful of incidents is easily dominated by one, and the reader must be able to see that.
- **A bucket with no incidents** is a gap (`y: null`, "No incidents"), never 0 minutes.
- **Urgency split.** When the API separates high- and low-urgency incidents, each metric shows one series per urgency, with high urgency first.

## Incident Sources

A pie chart shows the range's incidents by incident source — the PagerDuty integration or origin that triggered them — using the API's counts per source. Hovering a slice shows the source, its count, and the API's share. When the API groups small sources as "Other", the pie shows that slice; the UI does not merge sources itself. Selecting services in the breakdown table limits the pie to those services.

## Breakdown

Breakdown items are PagerDuty services, labeled by service name. Default sort: MTTR descending.

## Layers and Registration

| Layer | Specific contents |
|---|---|
| `domain/` | `SupportHealthMetricId`, the metric table, urgency labels |
| `application/` | `SupportHealthRepository` port; `loadOverview`; `toChartSpecs` |
| `infrastructure/` | Adapter over the generated client; maps "no incidents" to a missing value |
| `ui/` | Page composed from Charting's dimension page kit |

Area descriptor: `id: 'support-health'`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, `usesDateRange: true`. Its read routes are added to `DEMO_READ_ROUTES`.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| No incidents in a bucket | Gap labeled "No incidents" | 0 duration | A mean over zero incidents does not exist; 0 would read as instant response. |
| Showing incident counts | Its own metric, and in MTTA/MTTR hovers | Hover only | Volume matters on its own, and means over small samples mislead without the sample size beside them. |
| Incident sources view | Pie chart of the range's incidents by source | Stacked bars per bucket | The question is "where do incidents come from" over the period; a part-to-whole view answers it at a glance. |

## Open Questions & Future Decisions

### Deferred
1. Whether the API serves urgency splits, per-source counts, and the with/without-notes split.
2. Whether percentiles (e.g. p90 time to resolve) will be added beside the means.

## References

- `docs/intent/code-quality/code-quality-design.md` — reference shape for dimensions
- `docs/intent/charting/charting-design.md`
- `docs/high-level-design.md` — Domain: reporting dimensions and metrics
