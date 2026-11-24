# Test Coverage — EARS Specs

Design: `test-coverage-design.md`

## Metrics

- [ ] **TC-001**: The Test Coverage dimension shall present `tc.line-coverage` "Line Coverage (GitHub)" (percent, higher is better, snapshot), and, when the API provides it, `tc.branch-coverage` "Branch Coverage (GitHub)" (percent, higher is better, snapshot).
- [ ] **TC-002**: The Test Coverage dimension shall show tenant-level values, per-repository values, and change over the range only as returned by the API, and shall never combine repositories' coverage in the browser.
- [ ] **TC-003**: When the API reports that a repository published no coverage report, the Test Coverage page shall show "No coverage reported" for it, never 0%.
- [ ] **TC-004**: When the API returns a metric id the Test Coverage dimension does not define, the Test Coverage adapter shall ignore it.

## Page

- [ ] **TC-PAGE-001**: The Test Coverage page shall be built from Charting's dimension page kit, with GitHub repositories, labeled `owner/name`, as its source items.
- [ ] **TC-PAGE-002**: The Test Coverage source-item table shall be initially sorted by Line Coverage, ascending.
- [ ] **TC-PAGE-003**: When the tenant's configuration collects no GitHub repositories, the Test Coverage page shall show "No GitHub repositories are configured for this tenant" in place of the panels.

## Registration

- [ ] **TC-REG-001**: The Test Coverage area descriptor shall have id `test-coverage`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, and `usesDateRange: true`.
- [ ] **TC-REG-002**: The Test Coverage read routes shall be listed in the API client's `DEMO_READ_ROUTES`.
