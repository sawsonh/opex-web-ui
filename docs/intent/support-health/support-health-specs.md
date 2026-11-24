# Support Health — EARS Specs

Design: `support-health-design.md`

## Metrics

- [ ] **SH-001**: The Support Health dimension shall present `sh.incident-count` "Incident Count" (count, lower is better, flow), `sh.incidents-with-notes` "Incidents with Notes" (count, neutral, flow), `sh.mtta` "MTTA (Mean Time to Acknowledge)" (duration, lower is better, range), and `sh.mttr` "MTTR (Mean Time to Resolve)" (duration, lower is better, range), where "range" tiles show the API's value over the whole date range.
- [ ] **SH-002**: The Support Health page shall draw Incident Count as bars per bucket, and its tile shall show the API's total for the range.
- [ ] **SH-003**: The Support Health page shall draw Incidents with Notes as stacked bars per bucket, with notes and without, using the API's counts, and shall not compute a share of incidents with notes.
- [ ] **SH-004**: When the API provides a bucket's incident count, the MTTA and MTTR chart hovers shall show it ("mean of {n} incidents").
- [ ] **SH-005**: When a bucket has no incidents, the MTTA and MTTR charts shall show a gap labeled "No incidents", never zero.
- [ ] **SH-006**: When the API separates high- and low-urgency incidents, each Support Health metric shall show one series per urgency, high urgency first.
- [ ] **SH-007**: When the API returns a metric id the Support Health dimension does not define, the Support Health adapter shall ignore it.

## Incident sources

- [ ] **SH-SRC-001**: The Support Health page shall show a pie chart of the range's incidents by incident source, using the API's count and share per source.
- [ ] **SH-SRC-002**: The Support Health incident-source pie shall show an "Other" slice only when the API returns one, and shall never merge sources itself.
- [ ] **SH-SRC-003**: While PagerDuty services are checked for comparison, the Support Health incident-source pie shall show only those services' incidents, as returned by the API.

## Page

- [ ] **SH-PAGE-001**: The Support Health page shall be built from Charting's dimension page kit, with PagerDuty services, labeled by service name, as its source items.
- [ ] **SH-PAGE-002**: The Support Health source-item table shall be initially sorted by MTTR, descending.

## Registration

- [ ] **SH-REG-001**: The Support Health area descriptor shall have id `support-health`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, and `usesDateRange: true`.
- [ ] **SH-REG-002**: The Support Health read routes shall be listed in the API client's `DEMO_READ_ROUTES`.
