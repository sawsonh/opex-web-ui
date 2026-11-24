# App Shell — EARS Specs

Design: `app-shell-design.md`

## Composition

- [ ] **SHELL-COMP-001**: When the application starts, the composition root shall, in order, parse the profile configuration, create the sign-in adapter and wait for session restore and tab handoff to settle, create the API client and query client, register the area descriptors, and render the router.
- [ ] **SHELL-COMP-002**: The App Shell shall build navigation and area routes only from the area descriptors listed in `src/app/areas.ts`, and shall import no area module's internals.
- [ ] **SHELL-COMP-003**: The App Shell shall load each area's component lazily, on first navigation to that area.

## Routes

- [ ] **SHELL-ROUTE-001**: While the session is in the `none` state, `/` shall show the landing page with "View demo", "Sign in", and "Request access".
- [ ] **SHELL-ROUTE-002**: While the session is in the `signedIn` state, `/` shall redirect to the user's home.
- [ ] **SHELL-ROUTE-003**: The user's home shall be `/t/{activeTenant}` for a user with a role and at least one tenant, and the "no access yet" page otherwise.
- [ ] **SHELL-ROUTE-004**: When `/t/{tenantId}` or `/demo/{tenantId}` is opened without an area, the App Shell shall redirect to the first area the session may see for that tenant.
- [ ] **SHELL-ROUTE-005**: When a route that requires the `signedIn` state is opened in the `none` or `demo` state, the App Shell shall navigate to `/sign-in` with that route as return-to.
- [ ] **SHELL-ROUTE-006**: When a route matches no defined route, the App Shell shall show a not-found page with a link home.
- [ ] **SHELL-ROUTE-007**: When a route requires a capability the session lacks for the route's tenant, the App Shell shall show a not-permitted page and omit the area from navigation.

## Demo routes

- [ ] **SHELL-DEMO-001**: When `/demo` is opened and the demo-tenant list has exactly one tenant, the App Shell shall redirect to `/demo/{tenantId}` for that tenant.
- [ ] **SHELL-DEMO-002**: When `/demo` is opened and the demo-tenant list has several tenants, the App Shell shall show a picker listing them by name in the API's order.
- [ ] **SHELL-DEMO-003**: When `/demo` is opened and the demo-tenant list is empty, the App Shell shall show "No demo is available right now" with "Sign in" and "Request access".
- [ ] **SHELL-DEMO-004**: When `/demo` is opened after a demo read was rejected, the App Shell shall also show "That demo is no longer available".
- [ ] **SHELL-DEMO-005**: When a `/demo/{tenantId}` route names a tenant not in the demo-tenant list, the App Shell shall show a not-found page and send no reads for that tenant.
- [ ] **SHELL-DEMO-006**: While the session is in the `demo` state, the App Shell shall show only areas whose descriptor has `availableInDemo: true`.
- [ ] **SHELL-DEMO-007**: While the session is in the `demo` state, the App Shell shall show a non-dismissible bar reading "You're viewing a demo" with "Request access" and "Sign in", and shall show no user menu.
- [ ] **SHELL-DEMO-008**: While the session is in the `demo` state and there are several demo tenants, the tenant switcher shall list only the demo tenants.

## Tenant selection

- [ ] **SHELL-TENANT-001**: When a `/t/` or `/demo/` route's `{tenantId}` is not 24 lowercase hexadecimal characters, the App Shell shall show the not-found page and send no request.
- [ ] **SHELL-TENANT-002**: When a signed-in editor opens a `/t/{tenantId}` route for a tenant not in their listed tenants, the App Shell shall show a not-permitted page and send no request for that tenant.
- [ ] **SHELL-TENANT-003**: When the administrator opens a `/t/{tenantId}` route for a tenant not in the loaded tenant list, the App Shell shall reload the tenant list once, and only if the tenant is still absent show a not-permitted page and send no further request for that tenant.
- [ ] **SHELL-TENANT-004**: While the administrator is signed in, the App Shell shall load the API's tenant list once per session as the administrator's tenant choices.
- [ ] **SHELL-TENANT-005**: If the administrator's tenant list fails to load, then `/t/` routes shall show the failure with a retry, while `/admin/access-requests` and `/admin/tenants` remain usable.
- [ ] **SHELL-TENANT-006**: The App Shell shall show each tenant by its name from the API, and by its tenant id until the name has loaded or if it fails to load.
- [ ] **SHELL-TENANT-007**: While the session is `signedIn` and the active tenant's type is known to be `DEMO`, the top bar shall show a "Public demo" badge beside the tenant name, on every area of that tenant, explaining that anyone can view its reporting data and insights without signing in; while the type is unknown, it shall show no badge and block nothing.
- [ ] **SHELL-TENANT-008**: While the session is `signedIn`, the App Shell shall show the tenant switcher only when the user has more than one tenant choice, as a searchable list.
- [ ] **SHELL-TENANT-009**: When the user switches tenant, in the `signedIn` or `demo` state, the App Shell shall keep the current area and date range.
- [ ] **SHELL-TENANT-010**: When a signed-in user arrives at home, the App Shell shall choose as active tenant the tenant last used in this browser if the user may still see it, and otherwise the first of the user's tenant choices.
- [ ] **SHELL-TENANT-011**: If reading or writing the last-used tenant in `localStorage` fails, then the App Shell shall fall back to the first tenant choice without error.

## Date range

- [ ] **SHELL-RANGE-001**: The date range domain logic shall resolve presets against today's calendar date in the browser's time zone: `last-7-days` to today − 6 through today, `last-30-days` to today − 29 through today, `last-90-days` to today − 89 through today, `last-quarter` to the first through last day of the previous calendar quarter, and `year-to-date` to 1 January of the current year through today, all inclusive.
- [ ] **SHELL-RANGE-002**: The date range domain logic shall have no dependency on React or browser APIs other than receiving today's date as input.
- [ ] **SHELL-RANGE-003**: While an area whose descriptor has `usesDateRange: true` is shown, the top bar shall show the date range control, and the range shall be read from and written to the query string as `range={preset}` or `from={date}&to={date}`.
- [ ] **SHELL-RANGE-004**: When the query string has no range, the App Shell shall use `last-30-days`.
- [ ] **SHELL-RANGE-005**: The App Shell shall display the resolved start and end dates beside a preset's name.
- [ ] **SHELL-RANGE-006**: When a link with a preset range is opened, the App Shell shall resolve the preset against the current date rather than the date the link was made.
- [ ] **SHELL-RANGE-007**: If the query string's range is unparsable, or a custom range has `from` after `to` or `to` after today, then the App Shell shall replace it with the default range and tell the user the link's range was not valid.
- [ ] **SHELL-RANGE-008**: When the user changes the date range, the App Shell shall push a browser history entry.
- [ ] **SHELL-RANGE-009**: The App Shell shall display dates and times received from the API in the browser's time zone.

## Layout

- [ ] **SHELL-LAYOUT-001**: Where the profile is `LOCAL` or `QA`, the top bar shall show the profile name beside the logo; where the profile is `PROD`, it shall not.
- [ ] **SHELL-LAYOUT-002**: While the session is `signedIn`, the user menu shall show the user's email and "Sign out", and also "Change password" for Cognito sessions.
- [ ] **SHELL-LAYOUT-003**: When the route changes, the App Shell shall set the document title to "{area} · {tenant} · OpEx" and move focus to the page's main heading.

## Failure containment

- [ ] **SHELL-FAIL-001**: If an area throws while rendering, then the App Shell shall show "This section failed to load" with a retry in place of that area, keeping the top bar and navigation working.
- [ ] **SHELL-FAIL-002**: If an area's code chunk fails to load, then the App Shell shall show "A new version of OpEx is available" with a reload button.
- [ ] **SHELL-FAIL-003**: The App Shell shall provide shared presentational components for an `ApiError` of kind `forbidden` ("Your account isn't permitted to do this"), `refused` ("Something went wrong", logged without request details), and `network` or `server` ("Couldn't load this data" with retry), for areas to render in place of the failed data.
