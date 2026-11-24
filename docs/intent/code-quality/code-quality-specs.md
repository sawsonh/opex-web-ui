# Code Quality — EARS Specs

Design: `code-quality-design.md`

## Metrics

- [ ] **CQ-001**: The Code Quality dimension shall present five metrics with these labels, units, directions, and aggregation: `cq.bugs` "Bugs" (count, lower is better, snapshot), `cq.code-smells` "Code Smells" (count, lower, snapshot), `cq.vulnerabilities` "Vulnerabilities" (count, lower, snapshot), `cq.duplication-density` "Duplication Density" (percent, lower, snapshot), and `cq.code-coverage` "Code Coverage (SonarQube)" (percent, higher, snapshot).
- [ ] **CQ-002**: The Code Quality dimension shall show tenant-level values, per-project values, and change over the range only as returned by the API, and shall never sum, average, or subtract them in the browser.
- [ ] **CQ-003**: When the API returns a metric id the Code Quality dimension does not define, the Code Quality adapter shall ignore it.
- [ ] **CQ-004**: When the API provides a severity breakdown (blocker, critical, major, minor, info) for Bugs or Vulnerabilities, the Code Quality page shall show that metric as a stacked bar chart by severity; otherwise as a single series.

## Page

- [ ] **CQ-PAGE-001**: The Code Quality page shall be built from Charting's dimension page kit: metric tiles, one tenant-level chart panel per metric, a source-item table of SonarQube projects with comparison, the granularity control, the no-source-items state, and "Open RCA ticket".
- [ ] **CQ-PAGE-002**: The Code Quality source-item table shall list SonarQube projects, initially sorted by Bugs, descending.
- [ ] **CQ-PAGE-003**: When a SonarQube project is in the API's results but has no data in the range, the Code Quality source-item table shall list it with "No data".
- [ ] **CQ-PAGE-004**: When the tenant's configuration collects no SonarQube projects, the Code Quality page shall show "No SonarQube projects are configured for this tenant" in place of the panels.
- [ ] **CQ-PAGE-005**: When the user chooses "Open RCA ticket" on the Code Quality page, the `MetricRef` shall carry the metric id, tenant, and date range, and the SonarQube project and point when chosen from one.

## Registration

- [ ] **CQ-REG-001**: The Code Quality area descriptor shall have id `code-quality`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, and `usesDateRange: true`.
- [ ] **CQ-REG-002**: The Code Quality read routes shall be listed in the API client's `DEMO_READ_ROUTES`.
- [ ] **CQ-REG-003**: The Code Quality domain layer shall have no dependency on React, Plotly, or HTTP.
