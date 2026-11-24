# Auth & Session — EARS Specs

Design: `auth-session-design.md`

## Session states

- [ ] **AUTH-STATE-001**: While the tab's `sessionStorage` holds tokens for a known pool, the session shall be in the `signedIn` state, whatever the route.
- [ ] **AUTH-STATE-002**: While the tab holds no tokens and the route is under `/demo`, the session shall be in the `demo` state.
- [ ] **AUTH-STATE-003**: While the tab holds no tokens and the route is not under `/demo`, the session shall be in the `none` state.
- [ ] **AUTH-STATE-004**: When a tab in the `signedIn` state navigates to a route under `/demo`, the application shall redirect it to the user's signed-in home.
- [ ] **AUTH-STATE-005**: The auth module shall implement the API client's `SessionCredentials` port from the current session state.

## Sign-in (Cognito pools)

- [ ] **AUTH-SIGNIN-001**: When the user submits email and password on `/sign-in`, the Cognito adapter shall sign in to the main user pool using SRP.
- [ ] **AUTH-SIGNIN-002**: When the user submits email and password on `/admin/sign-in`, the Cognito adapter shall sign in to the admin user pool using SRP.
- [ ] **AUTH-SIGNIN-003**: When Cognito returns `NEW_PASSWORD_REQUIRED` during sign-in to either pool, the sign-in page shall ask for a new password and complete the challenge with it.
- [ ] **AUTH-SIGNIN-004**: When the admin user pool returns `MFA_SETUP` during sign-in, the sign-in page shall show a QR code rendered in the browser from an `otpauth://` URI (issuer "OpEx", account the user's email) together with the secret as text, and complete setup with the first code the user enters.
- [ ] **AUTH-SIGNIN-005**: When the admin user pool returns `SOFTWARE_TOKEN_MFA` during sign-in, the sign-in page shall ask for an authenticator code and complete the challenge with it.
- [ ] **AUTH-SIGNIN-006**: The sign-in flow shall send the TOTP secret only to Cognito and to no other service.
- [ ] **AUTH-SIGNIN-007**: If sign-in to either pool fails because the user is unknown, the password is wrong, or the user is disabled, then the sign-in page shall show the same message, "Email or password is incorrect", in every case.
- [ ] **AUTH-SIGNIN-008**: If Cognito throttles a sign-in, challenge, or password-reset attempt, then the page shall show "Too many attempts" and keep its submit button disabled with a 30-second countdown.
- [ ] **AUTH-SIGNIN-009**: If a TOTP or reset code is wrong or expired, then the page shall show "That code is not valid. Try again." and keep the user on the same step.
- [ ] **AUTH-SIGNIN-010**: If Cognito rejects a new password under the pool's password policy, then the page shall show the pool's policy message.
- [ ] **AUTH-SIGNIN-011**: If Cognito reports that the administrator has required a password reset, then the sign-in page shall direct the user to "Forgot password".
- [ ] **AUTH-SIGNIN-012**: When sign-in completes, the application shall navigate to the return-to path if it is a same-origin relative path beginning with `/` and not `//`, and otherwise to the user's home.
- [ ] **AUTH-SIGNIN-013**: While the tab is in the `signedIn` state, the `/sign-in` and `/admin/sign-in` pages shall show "You're signed in as {email}" with "Go to OpEx" and "Sign out" in place of the sign-in form.
- [ ] **AUTH-SIGNIN-014**: The sign-in pages shall link to the access-request form and offer no sign-up.

## Password self-service (Cognito pools)

- [ ] **AUTH-PWD-001**: When a user on `/sign-in/forgot` or `/admin/sign-in/forgot` submits an email, the Cognito adapter shall request a password-reset code for that pool, and the page shall then accept the emailed code and a new password.
- [ ] **AUTH-PWD-002**: While a user is signed in through a Cognito pool, `/account/password` shall let them change their password by entering the current and a new password.
- [ ] **AUTH-PWD-003**: Password and one-time-code inputs shall allow paste and set `autocomplete` to `current-password`, `new-password`, or `one-time-code` as appropriate.

## Tokens and storage

- [ ] **AUTH-TOKEN-001**: The Cognito adapter shall store tokens and the session's pool marker only in the tab's `sessionStorage`.
- [ ] **AUTH-TOKEN-002**: When the API client asks for an access token, the Cognito adapter shall return a current access token, refreshing it with the refresh token when it has expired, and forcing a refresh when `forceRefresh` is requested.
- [ ] **AUTH-TOKEN-003**: The auth module shall never log a token, place a token in a URL, or send a token anywhere other than Cognito, `/v1`, and same-origin tabs over the session channel.
- [ ] **AUTH-TOKEN-004**: When the application starts with tokens in `sessionStorage`, the Cognito adapter shall configure Amplify for the stored pool and restore the session.

## LOCAL adapter

- [ ] **AUTH-LOCAL-001**: Where `ProfileConfig.auth.kind` is `local`, the sign-in pages shall list the configured development identities by label, and choosing one shall start a session with that identity's token as the access token and its configured pool.
- [ ] **AUTH-LOCAL-002**: Where `ProfileConfig.auth.kind` is `local`, the application shall hide password change, password reset, and the forgot-password links.
- [ ] **AUTH-LOCAL-003**: Where `ProfileConfig.auth.kind` is `local`, if the local API rejects the session, then the application shall return to the LOCAL sign-in page with "This development token has expired; mint a new one with `dotnet user-jwts`."
- [ ] **AUTH-LOCAL-004**: The LOCAL adapter shall be loaded through a dynamic import that is eliminated from bundles whose `ProfileConfig.auth.kind` is `cognito`.

## User context

- [ ] **AUTH-CLAIMS-001**: When the session comes from the main user pool or a LOCAL identity whose pool is `main`, the claims adapter shall set the role to `opex-editor` if the access token's `cognito:groups` contains `opex-editor`, and to no role otherwise.
- [ ] **AUTH-CLAIMS-002**: When the session comes from the main user pool or a LOCAL identity whose pool is `main`, the claims adapter shall set the tenants to every `tenant-{tenantId}` group in `cognito:groups` whose id is 24 lowercase hexadecimal characters, in group order without duplicates, ignoring other `tenant-` groups.
- [ ] **AUTH-CLAIMS-003**: When the session comes from the admin user pool or a LOCAL identity whose pool is `admin`, the claims adapter shall set the role to `opex-admin` and the tenants to all tenants, whatever tenant groups the token holds.
- [ ] **AUTH-CLAIMS-004**: The claims adapter shall set `userId` from the access token's `sub`, and `email` from the ID token for Cognito sessions or from the identity's label for LOCAL sessions.
- [ ] **AUTH-CLAIMS-005**: The claims adapter shall decode token payloads without verifying signatures, and no module outside `platform/auth` shall read Cognito claim names.
- [ ] **AUTH-CLAIMS-006**: While a main-pool session has no role or no tenants, the application shall show "Your account has no access yet" with a link to request access, and send no tenant-scoped requests.

## Capabilities

- [ ] **AUTH-CAP-001**: The `can(user, capability, tenant)` function shall grant `viewReporting` and `viewInsights` to the demo state only for the tenant named in the current `/demo/{tenantId}` route, to an editor for their listed tenants, and to the administrator for any tenant.
- [ ] **AUTH-CAP-002**: The `can` function shall grant `viewImprovement`, `editTenantConfiguration`, `requestInsightsRun`, `giveAnomalyFeedback`, and `writeImprovementTickets` to an editor for their listed tenants and to the administrator for any tenant, and never to the demo state.
- [ ] **AUTH-CAP-003**: The `can` function shall grant `readAccessRequests` and `manageTenants` only to the administrator.
- [ ] **AUTH-CAP-004**: The `can` function shall grant `submitAccessRequest` in every session state.

## Tabs

- [ ] **AUTH-TAB-001**: When a tab starts with no tokens, it shall post a `session-request` with a fresh nonce on the `opex-session` `BroadcastChannel` and wait up to 300 ms for a `session-offer` carrying that nonce.
- [ ] **AUTH-TAB-002**: While a tab is in the `signedIn` state, it shall answer a `session-request` with a `session-offer` carrying the request's nonce, the session's pool, and its `sessionStorage` session entries.
- [ ] **AUTH-TAB-003**: When a waiting tab receives the first `session-offer` carrying its nonce, it shall write the offered entries to its `sessionStorage` and restore the session; it shall ignore offers carrying any other nonce.
- [ ] **AUTH-TAB-004**: If no matching `session-offer` arrives within 300 ms, then the tab's session state shall follow its route (`none` or `demo`).
- [ ] **AUTH-TAB-005**: While the tab handoff is pending, the application shall show a neutral loading state and not render the landing page.
- [ ] **AUTH-TAB-006**: When a user signs in, the application shall not change the state of other tabs.

## Sign-out and session end

- [ ] **AUTH-END-001**: When the user signs out and the unsaved-changes guard allows it, the auth module shall revoke the refresh token with Cognito, clear the tab's tokens even if revocation fails, delete all form drafts in the tab, post `signed-out` on the session channel, and go to the landing page.
- [ ] **AUTH-END-002**: When a tab receives `signed-out` on the session channel, it shall clear its tokens, delete its form drafts, and go to the landing page, without prompting even if it has unsaved changes.
- [ ] **AUTH-END-003**: When `sessionRejected()` is called or a token refresh fails in a Cognito session, the auth module shall clear the tab's tokens, keep form drafts, and navigate to the sign-in route of the session's pool with "Your session has ended. Sign in again." and the current route as return-to.
- [ ] **AUTH-END-004**: When `demoRejected()` is called (a demo read returned 401), the application shall navigate to `/demo` with "That demo is no longer available", staying in the demo state.
- [ ] **AUTH-END-005**: When the API answers `403`, the auth module shall keep the session.
