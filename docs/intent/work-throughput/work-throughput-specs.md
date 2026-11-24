# Work Throughput — EARS Specs

Design: `work-throughput-design.md`

## Metrics

- [ ] **WT-001**: The Work Throughput dimension shall present `wt.tickets-closed` "Tickets Closed" (count, neutral, flow) and `wt.wip-cycle-time` "WIP Cycle Time" (duration, lower is better, range — its tile shows the API's value over the whole date range).
- [ ] **WT-002**: The Work Throughput page shall draw Tickets Closed as bars per bucket, and its tile shall show the API's total for the range.
- [ ] **WT-003**: When the API provides Tickets Closed by issue type, the Work Throughput page shall show it as a stacked bar chart by issue type.
- [ ] **WT-004**: The Work Throughput page shall label WIP Cycle Time with the statistic the API names for it, and show the bucket's ticket count in the hover when the API provides it.
- [ ] **WT-005**: When the API returns a metric id the Work Throughput dimension does not define, the Work Throughput adapter shall ignore it.

## Page

- [ ] **WT-PAGE-001**: The Work Throughput page shall be built from Charting's dimension page kit, with Jira projects, labeled by key and name, as its source items.
- [ ] **WT-PAGE-002**: The Work Throughput source-item table shall be initially sorted by Tickets Closed, descending.

## Registration

- [ ] **WT-REG-001**: The Work Throughput area descriptor shall have id `work-throughput`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, and `usesDateRange: true`.
- [ ] **WT-REG-002**: The Work Throughput read routes shall be listed in the API client's `DEMO_READ_ROUTES`.
