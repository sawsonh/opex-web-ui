---
parent: high-level-design
prefix: API
---

# API Client

## Context and Design Philosophy

Every read and write the UI makes goes to `opex-web-api` under `/v1`. The API Client is the one path those requests take. It owns four things: the generated contract, the transport (base path, credentials, timeouts), the local request guards that mirror the API's rules, and the mapping of HTTP failures into a small set of typed errors that the rest of the UI handles uniformly.

Principles:

- **The OAS is the contract.** Request and response types are generated, never hand-written. Module `infrastructure` adapters call the generated client and translate to domain types; the API Client itself knows no domain.
- **The client never decides who the user is.** It asks a `SessionCredentials` port, implemented by Auth & Session, what kind of session the tab has and for a current access token.
- **Guards fail closed, locally.** A request the current session must not send is refused before it reaches the network. The API remains the authority; the guard exists so a UI bug cannot send it.
- **Errors are values the UI can switch on.** Callers handle an `ApiError` union, not status codes.

## Contract Generation

- A snapshot of the `opex-web-api` OAS is committed at `openapi/opex-web-api.json`. A script (`npm run api:update`) replaces it from the API's published spec; updating the snapshot is a reviewed change like any other.
- `openapi-typescript` generates `src/platform/api/generated/schema.d.ts` from the snapshot. Generation runs before type-check in the Build stage and in local `typecheck`/`dev` scripts; the generated file is git-ignored so it cannot drift from the snapshot.
- `openapi-fetch` provides the typed client over the generated `paths`. Only `platform/api` and module `infrastructure` layers import the generated types (lint-enforced).

## Transport

- One `openapi-fetch` client instance, created by the composition root with the `/v1` base path on the page's own origin. No absolute URL exists in any profile.
- Every request carries an `AbortSignal` combining the caller's signal (from TanStack Query) with a 30-second timeout.
- Responses are JSON. A non-JSON error body is treated as having no detail.

## Credentials by Request

The `SessionCredentials` port:

```ts
interface SessionCredentials {
  mode(): 'none' | 'demo' | 'signedIn';
  accessToken(opts?: { forceRefresh?: boolean }): Promise<string>; // signedIn only
  tenantScope(): { kind: 'all' } | { kind: 'listed'; tenants: TenantId[] }; // signedIn only
  sessionRejected(): void;   // the API rejected the signed-in session after a refresh
  demoRejected(): void;      // the API answered a demo read with 401
}
```

| Request | Session mode | `Authorization` header |
|---|---|---|
| `POST /v1/access-requests` (public submission) | any | none |
| Demo read (see guard) | `demo` | none |
| Any other permitted request | `signedIn` | `Bearer {access token}` |

The ID token is never requested from the port and never sent. The token value never appears in logs, error objects, or thrown messages.

## Request Guards

Guards run in a request middleware before the network call. A refused request rejects with `ApiError { kind: 'refused', reason }` and no network traffic.

| Session mode | Allowed | Everything else |
|---|---|---|
| `none` | `POST /v1/access-requests` | refused (`reason: 'no-session'`) |
| `demo` | `POST /v1/access-requests`; `GET` whose path is under `/v1/tenants/{demoTenantId}/` and matches an entry in `DEMO_READ_ROUTES` | refused (`reason: 'demo'`) |
| `signedIn`, scope `listed` | Requests whose path names no tenant, or names a tenant in the listed tenants | refused (`reason: 'tenant'`) |
| `signedIn`, scope `all` | All requests | — |

`DEMO_READ_ROUTES` is an explicit allowlist of OAS path templates for reporting-dimension and insights reads, kept in `platform/api` beside the guard. A new dimension's demo-readable routes are added to it in the same change that adds the dimension. Path matching is done on the template the generated client was called with, not on a string search of the final URL, so encoded or crafted segments cannot widen it.

## 401 Handling

```
signedIn request ──▶ 401 ──▶ accessToken({forceRefresh:true}) ──▶ retry once
                                     │ fails                          │ 401 again
                                     ▼                                ▼
                          sessionRejected()                 sessionRejected()
                          reject: unauthorized              reject: unauthorized

demo request ──▶ 401 ──▶ demoRejected(); reject: unauthorized
```

- The retry replays a clone of the original request, taken before the first send, so request bodies survive.
- Concurrent requests that receive 401 share one forced refresh rather than each refreshing.
- What `sessionRejected` and `demoRejected` do (end the session, route to sign-in) is Auth & Session's.

## Error Mapping

| Condition | `ApiError.kind` | Carries |
|---|---|---|
| Local guard refusal | `refused` | `reason` |
| 401 after the retry, or any demo 401 | `unauthorized` | — |
| 403 | `forbidden` | API detail message if present |
| 404 | `notFound` | — |
| 409 | `conflict` | API detail |
| 400, 422 | `validation` | API detail and per-field messages when the body provides them |
| 429 | `rateLimited` | `retryAfterSeconds` when the header is present |
| 5xx | `server` | status |
| Fetch rejected, timeout | `network` | `timedOut: boolean` |
| Caller aborted | `aborted` | — |

Area-specific meanings of an error (for example, insights reporting the paused state through a refused run) are mapped by that area's `infrastructure` adapter from the `ApiError` and its detail; the API Client stays domain-free.

## Query Defaults

The API Client exports the TanStack Query defaults derived from its error classification, applied by the composition root:

- Queries retry up to 2 times, with backoff, only on `network`, `server`, and `rateLimited` (honoring `retryAfterSeconds`); never on other kinds.
- Mutations never retry automatically. Retry is a user action (see Forms).

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| OAS source in the build | Committed snapshot, types generated and git-ignored | Fetch the OAS at build time; commit generated types | A snapshot makes contract changes reviewable diffs and keeps Build free of network and cross-repo credentials; generating rather than committing types removes a second artifact that can drift. |
| How the client learns the session | `SessionCredentials` port implemented by Auth & Session, wired by the composition root | `platform/api` importing `platform/auth` directly | Keeps the two platform modules independent and lets tests drive every session mode with a fake. |
| Demo guard shape | Explicit allowlist of path templates plus `GET` and demo-tenant checks | Any `GET` under the demo tenant | An allowlist cannot silently admit a future sensitive read (e.g. improvement tickets) just because it is a `GET`. |
| Tenant containment | Local guard on the tenant path segment for `listed` scope | Rely on the API's 403 | The HLD's tenant-containment metric requires the UI never to send such a request; the API's 403 remains the real enforcement. |
| 401 recovery | One forced refresh, one replay, shared across concurrent requests | No retry; retry per request | Covers clock skew and early revocation without a refresh storm; a second 401 means the session is genuinely gone. |
| Timeout | 30 seconds per request | Browser default (none) | A hung request otherwise leaves a panel loading forever; 30 s comfortably exceeds normal API latency. |

## Open Questions & Future Decisions

### Deferred
1. Where the API publishes its OAS for `api:update` (URL or artifact), to be fixed with `opex-web-api`.
2. The API's error body shape (e.g. RFC 9457 problem details) and how field-level validation messages are expressed; the `validation` mapping follows it.
3. Whether the API accepts a client-generated request id header for correlating UI errors with API logs.

## References

- `docs/high-level-design.md` — The OAS is the contract; Sessions; Key Design Decisions 7, 10
- `docs/intent/auth-session/auth-session-design.md` — implements `SessionCredentials`
- `docs/intent/profile-config/profile-config-design.md` — `demoTenantId`
- `opex-web-api` — OpenAPI Specification
