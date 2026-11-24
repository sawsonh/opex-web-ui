# Development Velocity — EARS Specs

Design: `development-velocity-design.md`

## Metrics

- [ ] **DV-001**: The Development Velocity dimension shall present `dv.pr-cycle-time` "PR Cycle Time", `dv.pr-time-to-first-engagement` "Time to First Engagement", and `dv.pr-first-engagement-to-merge` "First Engagement → Merge" (each duration, lower is better, range), `dv.pr-complexity` "PR Complexity" (count, lower is better, range), and `dv.pr-complexity-comment-ratio` "Complexity vs. Comments" (ratio, neutral, range), where "range" tiles show the API's value over the whole date range.
- [ ] **DV-002**: The Development Velocity page shall label each series with the statistic the API names for it (for example "PR Cycle Time — median"), and shall never compute a statistic from per-pull-request data.
- [ ] **DV-003**: When the API provides a bucket's pull-request count, the Development Velocity chart hover shall show it as the sample size ("based on {n} PRs").
- [ ] **DV-004**: When the API returns a metric id the Development Velocity dimension does not define, the Development Velocity adapter shall ignore it.

## Per-pull-request scatter

- [ ] **DV-PR-001**: The Development Velocity page shall show a scatter chart with one point per pull request merged in the range, complexity on the x axis and review comments on the y axis.
- [ ] **DV-PR-002**: Each point in the Development Velocity scatter chart shall show in its hover the repository, the pull request's number and title, complexity, comments, and cycle time.
- [ ] **DV-PR-003**: When the API provides a pull request's GitHub URL, clicking its point in the Development Velocity scatter chart shall offer "Open in GitHub" as a plain link in a new tab.
- [ ] **DV-PR-004**: While repositories are checked for comparison, the Development Velocity scatter chart shall plot only those repositories' pull requests, colored by repository.
- [ ] **DV-PR-005**: When the API returns fewer pull requests than the total it reports for the range, the Development Velocity scatter panel shall state how many are plotted out of the total.

## Page

- [ ] **DV-PAGE-001**: The Development Velocity page shall be built from Charting's dimension page kit, with GitHub repositories, labeled `owner/name`, as its source items.
- [ ] **DV-PAGE-002**: The Development Velocity source-item table shall be initially sorted by PR Cycle Time, descending.

## Registration

- [ ] **DV-REG-001**: The Development Velocity area descriptor shall have id `development-velocity`, group `dimensions`, capability `viewReporting`, `availableInDemo: true`, and `usesDateRange: true`.
- [ ] **DV-REG-002**: The Development Velocity read routes, including the per-pull-request list, shall be listed in the API client's `DEMO_READ_ROUTES`.
