# API Client — EARS Specs

Design: `api-client-design.md`

## Contract

- [ ] **API-CONTRACT-001**: The build shall generate the API's TypeScript types from the committed OAS snapshot `openapi/opex-web-api.json` before type-checking, and the generated file shall not be committed.
- [ ] **API-CONTRACT-002**: The lint configuration shall reject imports of the generated API types from any module other than `platform/api` and area `infrastructure` layers.

## Transport

- [ ] **API-TRANS-001**: The API client shall send every request to the `/v1` base path on the page's own origin.
- [ ] **API-TRANS-002**: The API client shall abort any request that has not completed 30 seconds after it was sent, and reject it as `network` with `timedOut: true`.
- [ ] **API-TRANS-003**: When the caller's abort signal fires, the API client shall abort the request and reject it as `aborted`.

## Credentials

- [ ] **API-AUTHZ-001**: When the session mode is `signedIn` and the request is neither `POST /v1/access-requests` nor `GET /v1/demo-tenants`, the API client shall send `Authorization: Bearer` with the access token obtained from `SessionCredentials`.
- [ ] **API-AUTHZ-002**: The API client shall send `POST /v1/access-requests` and `GET /v1/demo-tenants` without an `Authorization` header in every session mode.
- [ ] **API-AUTHZ-003**: When the session mode is `demo`, the API client shall send every permitted request without an `Authorization` header.
- [ ] **API-AUTHZ-004**: The API client shall never send an ID token, and shall never include a token value in a log entry, an error object, or an error message.

## Guards

- [ ] **API-GUARD-001**: If a request's `{tenantId}` path parameter is not 24 lowercase hexadecimal characters, then the API client shall refuse it with `ApiError` kind `refused`, reason `tenant`, in every session mode, without sending it.
- [ ] **API-GUARD-002**: When the session mode is `none`, the API client shall refuse every request other than `POST /v1/access-requests` and `GET /v1/demo-tenants` with reason `no-session`, without sending it.
- [ ] **API-GUARD-003**: When the session mode is `demo`, the API client shall allow only `POST /v1/access-requests`, `GET /v1/demo-tenants`, and `GET` requests whose OAS path template is in `DEMO_READ_ROUTES` and whose `{tenantId}` is in the demo-tenant list, and shall refuse every other request with reason `demo`, without sending it.
- [ ] **API-GUARD-004**: When the session mode is `demo`, the API client shall match a request against `DEMO_READ_ROUTES` by the OAS path template the generated client was called with, not by inspecting the final URL string.
- [ ] **API-GUARD-005**: When the session mode is `signedIn` and the tenant scope is `listed`, the API client shall refuse, with reason `tenant` and without sending it, any request whose path names a tenant not in the listed tenants.
- [ ] **API-GUARD-006**: When the session mode is `signedIn` and the tenant scope is `all` (the administrator), the API client shall not refuse requests on tenant grounds other than a malformed tenant id.

## Demo-tenant list

- [ ] **API-DEMO-001**: The API client shall provide `demoTenants()`, which fetches `GET /v1/demo-tenants` at most once per page load until invalidated, and returns its items in the order the API returned them.
- [ ] **API-DEMO-002**: When `GET /v1/demo-tenants` returns an item whose `tenantId` is not 24 lowercase hexadecimal characters, the API client shall drop that item from the demo-tenant list.
- [ ] **API-DEMO-003**: When the session mode is `demo` and a request needs the demo-tenant check before the list has loaded, the API client shall load the demo-tenant list first and then apply the check.
- [ ] **API-DEMO-004**: When `invalidateDemoTenants()` is called, the API client shall discard the cached demo-tenant list so the next `demoTenants()` call fetches it again.

## 401 handling

- [ ] **API-401-001**: When a `signedIn` request receives `401`, the API client shall obtain a token with `forceRefresh` and resend a clone of the original request once.
- [ ] **API-401-002**: When several `signedIn` requests receive `401` concurrently, the API client shall perform a single forced refresh shared by all of them.
- [ ] **API-401-003**: If the forced refresh fails, or the resent `signedIn` request receives `401` again, then the API client shall call `SessionCredentials.sessionRejected()` and reject the request as `unauthorized`.
- [ ] **API-401-004**: When a `demo` request receives `401`, the API client shall call `invalidateDemoTenants()`, call `SessionCredentials.demoRejected()`, and reject the request as `unauthorized` without retrying.

## Error mapping

- [ ] **API-ERR-001**: The API client shall map responses to `ApiError` kinds as follows: `403` to `forbidden`, `404` to `notFound`, `409` to `conflict`, `400` and `422` to `validation`, `429` to `rateLimited`, `5xx` to `server`, and a rejected fetch to `network`.
- [ ] **API-ERR-002**: When a `validation` response body provides per-field messages, the API client shall include them in the `ApiError`.
- [ ] **API-ERR-003**: When a `429` response carries a `Retry-After` header, the API client shall include its value in seconds as `retryAfterSeconds`.
- [ ] **API-ERR-004**: If an error response body is not JSON, then the API client shall map the error with no detail rather than failing.

## Query defaults

- [ ] **API-QUERY-001**: The API client's query defaults shall retry a failed query at most 2 times, with backoff, only when the error kind is `network`, `server`, or `rateLimited`.
- [ ] **API-QUERY-002**: When a query fails as `rateLimited` with `retryAfterSeconds`, the API client's query defaults shall wait at least that long before retrying.
- [ ] **API-QUERY-003**: The API client's mutation defaults shall not retry mutations automatically.
