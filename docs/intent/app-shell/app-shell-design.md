---
parent: high-level-design
prefix: SHELL
---

# App Shell

## Context and Design Philosophy

The App Shell is the frame every area renders inside: bootstrap and composition, routing, layout and navigation, the landing page, the demo indicator, the tenant switcher, and the date range that reporting views share. It also contains failures so that one area breaking does not take down the others.

Principles:

- **The URL is the state worth sharing.** Tenant, area, and date range live in the route, so any view can be linked and reloaded. Nothing that defines what the user is looking at lives only in memory.
- **The shell composes; areas own their content.** Areas register routes and navigation entries through a descriptor; the shell never imports an area's internals, and areas never import the shell.
- **Show what the user may use.** Navigation and routes are gated through `can()` from Auth & Session. Gating is presentation; the API enforces.

## Composition Root

`src/app/main.tsx` is the only place concrete implementations meet:

1. Parse `ProfileConfig`; on failure render the misconfiguration page and stop.
2. Create the auth adapter for `config.auth.kind`, wait for `restore()` and tab handoff to settle.
3. Create the API client with the `SessionCredentials` implementation, and the TanStack `QueryClient` with the API Client's query defaults.
4. Register area descriptors and render the router.

Each area module exports one descriptor:

```ts
type AreaDescriptor = {
  id: string;                        // e.g. 'code-quality', 'insights'
  group: 'dimensions' | 'insights' | 'improvement' | 'administration';
  label: string;
  path: string;                      // segment under the tenant route
  capability: Capability;            // gates nav and route
  availableInDemo: boolean;
  usesDateRange: boolean;
  load: () => Promise<{ default: ComponentType }>;  // lazy
};
```

The registry is one list in `src/app/areas.ts`. Adding a dimension adds a module and one entry there; no other module changes.

## Routes

| Route | State | Shows |
|---|---|---|
| `/` | none | Landing page: what OpEx is; "View demo", "Sign in", "Request access". Signed-in tabs are redirected to their home. |
| `/demo` | demo | Loads the demo-tenant list. One demo tenant: redirects to `/demo/{tenantId}`. Several: a picker listing them by name, in the API's order. None: "No demo is available right now" with "Sign in" and "Request access". Arriving here after a demo read was rejected adds "That demo is no longer available". |
| `/demo/{tenantId}` → `/demo/{tenantId}/{area}` | demo | A demo tenant's areas with `availableInDemo`; redirects to the first. A `tenantId` not in the demo-tenant list shows a not-found page and sends no reads. |
| `/sign-in`, `/admin/sign-in`, `/sign-in/forgot`, `/admin/sign-in/forgot` | none | Sign-in pages (Auth & Session) |
| `/request-access` | any | Access-request form (Access Requests) |
| `/t/{tenantId}` → `/t/{tenantId}/{area}` | signedIn | The tenant's areas the user may see. `/t/{tenantId}` redirects to the first permitted area. |
| `/admin/access-requests` | signedIn (admin) | Administrator's request list (Access Requests) |
| `/admin/tenants` | signedIn (admin) | Tenant list and tenant creation (Tenant Administration) |
| `/account/password` | signedIn (Cognito pools) | Change password (Auth & Session) |
| anything else | any | Not-found page with a link home |

- **Home** for a signed-in user is `/t/{activeTenant}`; for a main-pool user with no role or no tenants, the "no access yet" page.
- Demo routes live under `/demo/{tenantId}`, separate from `/t/{tenantId}`, so the session state is derivable from the route, and a demo link opens the same demo for any visitor. When there are several demo tenants, the top bar shows a demo-tenant switcher in place of the tenant switcher.
- Areas are lazy-loaded per route. Administrator-only areas are in chunks that other users never request.
- A route that requires `signedIn` opened in `none` or `demo` goes to `/sign-in` with the route as return-to.

## Tenant Selection

- The **active tenant** is the `{tenantId}` route segment. There is no other copy of it.
- A `{tenantId}` in a `/t/` or `/demo/` route that is not 24 lowercase hexadecimal characters shows the not-found page and sends no request.
- A route naming a tenant the user may not see (not in `listed` tenants; or, for the administrator, not in the API's tenant list after reloading that list once) shows a not-permitted page and sends no request for that tenant.
- Tenant choices: a main-pool user's `listed` tenants; for the administrator, the API's list of every tenant, loaded once per session. If that list fails to load, tenant routes show the failure with a retry, while `/admin/access-requests` and `/admin/tenants`, which need no active tenant, stay usable.
- Tenants are shown by name: from the administrator's tenant list, or for an editor from the API's details for each of their tenants. Until a name has loaded, or if it fails to load, the tenant id is shown.
- **Public demo marker**: when a signed-in user's active tenant is known to have type `DEMO`, the top bar shows a "Public demo" badge beside the tenant name, explaining that anyone can view this tenant's reporting data and insights without signing in. It stays visible on every area of that tenant. While the type is unknown (not loaded, or failed to load), no badge is shown and nothing is blocked.
- **Switcher**: shown only when the user has more than one tenant choice — or, in the demo, more than one demo tenant. It is a searchable list. Switching, signed in or in the demo, keeps the current area and date range.
- **Default tenant** on arriving at home: the tenant last used in this browser, if the user may still see it; otherwise the first tenant in the user's list. The last-used tenant is a per-browser convenience in `localStorage`, read and written defensively; losing it only changes the default.

## Date Range

Shown in the top bar for areas with `usesDateRange`, and carried in the query string so it is shared by every such area:

```
/t/6b0480d6e2f81b78584bc4d6/code-quality?range=last-30-days
/t/6b0480d6e2f81b78584bc4d6/code-quality?from=2026-08-01&to=2026-08-31
```

Pure domain logic in `shared/domain` (`DateRange`, `DateRangePreset`, `resolvePreset(preset, today)`), with no React or browser dependency.

| Preset id | Resolves to (inclusive, calendar dates) |
|---|---|
| `last-7-days` | today − 6 … today |
| `last-30-days` (default) | today − 29 … today |
| `last-90-days` | today − 89 … today |
| `last-quarter` | first … last day of the previous calendar quarter |
| `year-to-date` | 1 January of this year … today |
| custom | `from` … `to` as given |

- "Today" is the user's local calendar date, from the browser's time zone. Dates and times from the API (UTC) are displayed in that time zone. How a local date range is expressed in the API's query parameters follows the API contract.
- A preset link stays relative: opening `range=last-30-days` tomorrow shows tomorrow's last 30 days. The resolved dates are always displayed next to the preset name, so anyone can cite exactly what they saw.
- A custom range requires `from` ≤ `to` and `to` ≤ today. An invalid or unparsable range in the URL is replaced by the default and the user is told the link's range was not valid.
- Changing the range pushes a history entry, so Back returns to the previous view.

## Layout

```
┌────────────────────────────────────────────────────────────────────┐
│ OpEx  [QA]   Tenant ▾          Last 30 days (Jul 24–Aug 22) ▾      │ user@x ▾ │
├───────────────┬────────────────────────────────────────────────────┤
│ DIMENSIONS    │                                                    │
│  Code Quality │                                                    │
│  Test Cov.    │               area content                         │
│  Dev Velocity │                                                    │
│  Throughput   │                                                    │
│  Support      │                                                    │
│ Insights      │                                                    │
│ Improvement   │                                                    │
│ Settings      │  (editor, admin)                                   │
│ Access reqs   │  (admin)                                           │
└───────────────┴────────────────────────────────────────────────────┘
```

- **Demo indicator**: in the demo state, a persistent bar above the top bar reads "You're viewing a demo" with "Request access" and "Sign in". It cannot be dismissed. There is no user menu in the demo, and the tenant switcher lists only demo tenants (shown only when there are several).
- **Environment badge**: the profile name appears beside the logo in `LOCAL` and `QA`, never in `PROD`.
- **User menu**: email, "Change password" (Cognito pools), "Sign out".
- On every route change the document title is set to "{area} · {tenant} · OpEx" and focus moves to the page's main heading.

## Failure Containment

| Failure | Where it shows | What the user sees |
|---|---|---|
| Render error inside an area | Error boundary around the area outlet | "This section failed to load" with retry; top bar and navigation keep working |
| Lazy chunk fails to load | Same boundary, detected as a chunk-load error | "A new version of OpEx is available" with a reload button |
| `ApiError.forbidden` | The view that made the request | "Your account isn't permitted to do this" — the session continues |
| `ApiError.refused` | The view | Treated as a defect: "Something went wrong", logged without request details |
| `ApiError.network` / `server` after query retries | The view | "Couldn't load this data" with retry |
| Unknown route | Not-found page | Link home |

Each area renders its own data failures inside its panels; the shell provides the shared presentational components for these states so they look and read the same everywhere.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Where view state lives | Route path and query string | Global store; `sessionStorage` | Links must be shareable and reloadable; a single source avoids store/URL drift. |
| Demo route shape | `/demo/{tenantId}/{area}` | `/t/{tenantId}/{area}` in demo mode; `/demo/{area}` with the tenant held elsewhere | A distinct prefix keeps the demo state derivable from the route; the tenant id in the path makes demo links shareable when there are several demo tenants. |
| Area integration | Lazy descriptor list in `app/areas.ts` | Areas self-register via side-effect imports; shell imports area components directly | An explicit list is greppable and tree-shake friendly, and adding an area touches only that list and the new module. |
| Preset links | Stay relative, with resolved dates displayed | Rewrite presets to absolute dates in the URL | A "last 30 days" bookmark is useful precisely because it moves; showing the resolved dates keeps cited numbers unambiguous. |
| Calendar for "today" and displayed times | The browser's local time zone | UTC; a per-tenant time zone | Users read dates in their own day; the API's values stay UTC and the conversion is a presentation transform. |
| Tenant switch | Keep area and date range | Reset to the tenant's first area | Keeping the view lets a user compare the same metric across tenants. |

## Open Questions & Future Decisions

### Deferred
1. The earliest date with data, and whether the API's range parameters are calendar dates or UTC instants — affects the custom range's lower bound and how a local range is sent.
2. The endpoints for the administrator's tenant list and for an editor's tenant details (name, type).
3. Whether the insights area shares the top bar's date range or selects a report independently — follows the insights contract.

## References

- `docs/high-level-design.md` — Module structure; Sessions; Roles and tenants; Key Design Decision 6
- `docs/intent/auth-session/auth-session-design.md` — session states, `can()`, sign-in routes
- `docs/intent/api-client/api-client-design.md` — `ApiError`, query defaults
- `docs/intent/profile-config/profile-config-design.md` — `ProfileConfig`
