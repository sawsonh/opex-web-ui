---
parent: high-level-design
prefix: AUTH
---

# Auth & Session

## Context and Design Philosophy

Auth & Session decides what kind of session a browser tab has, gets and keeps the tokens for a signed-in session, and turns a token into the user context (role and tenants) the rest of the UI uses to decide what to show. It is the only module that touches Amazon Cognito, Amplify Auth, or Cognito claim names.

Principles:

- **Cognito authenticates; the API authorizes.** This module never verifies a token's signature and never treats its own reading of claims as permission. It reads claims only to decide what to show.
- **One port, swappable adapters.** Everything outside `platform/auth` talks to `AuthGateway` and `UserContext`. Amplify, the two pools, and the LOCAL development identities are adapter details.
- **Tokens are sensitive.** They live only in the tab's `sessionStorage`, travel only to Cognito, `/v1`, and same-origin tabs, and are never logged or put in a URL.

## Session States

A tab is in exactly one state:

| State | Determined by | Ends when |
|---|---|---|
| `signedIn` | Tokens present in the tab's `sessionStorage` (own or handed off) for a known pool | Sign-out (this or another tab), refresh failure, or the API rejecting the session after a refresh |
| `demo` | No tokens, and the current route is under `/demo` | Navigating out of `/demo`, or signing in |
| `none` | No tokens, and the route is not under `/demo` | Entering `/demo`, or signing in |

- `demo` and `none` are derived from the route, so they need no storage. `signedIn` takes precedence: a signed-in tab that navigates to a `/demo` route is redirected to its signed-in home rather than mixing a token-bearing session with demo reads.
- The module implements the API Client's `SessionCredentials` port from this state.

## The `AuthGateway` Port

```ts
type Pool = 'main' | 'admin';

interface AuthGateway {
  signIn(pool: Pool, email: string, password: string): Promise<SignInStep>;
  completeNewPassword(newPassword: string): Promise<SignInStep>;
  completeTotpSetup(code: string): Promise<SignInStep>;
  completeTotp(code: string): Promise<SignInStep>;
  changePassword(oldPassword: string, newPassword: string): Promise<void>;
  requestPasswordReset(pool: Pool, email: string): Promise<void>;
  confirmPasswordReset(pool: Pool, email: string, code: string, newPassword: string): Promise<void>;
  accessToken(opts?: { forceRefresh?: boolean }): Promise<string>;
  signOut(): Promise<void>;
  restore(): Promise<Session | null>;
}

type SignInStep =
  | { step: 'done'; session: Session }
  | { step: 'newPasswordRequired' }
  | { step: 'totpSetupRequired'; secret: string; otpauthUri: string }
  | { step: 'totpRequired' };
```

Failures reject with an `AuthError`:

| `AuthError.kind` | When | Shown to the user |
|---|---|---|
| `invalidCredentials` | Wrong password, unknown user, disabled user | "Email or password is incorrect" — identical for every cause, so the form reveals nothing about which accounts exist |
| `invalidCode` | Wrong or expired TOTP or reset code | "That code is not valid. Try again." |
| `passwordPolicy` | New password rejected by the pool's policy | The pool's policy message |
| `passwordResetRequired` | The administrator forced a reset | Directed to "Forgot password" |
| `tooManyAttempts` | Cognito throttled the attempt | "Too many attempts." The submit button stays disabled with a 30-second countdown before the next try. |
| `network` | Cognito unreachable | "Can't reach the sign-in service" with retry |

## Adapters

### Cognito adapter (QA, PROD)

- Built on Amplify Auth (`aws-amplify/auth`) using SRP. Amplify holds one pool configuration at a time; the adapter configures it for the pool chosen by the route before `signIn`, and for the stored pool before `restore`. A tab has at most one session, so one active configuration suffices.
- Amplify's token storage is set to a `sessionStorage`-backed key-value store. The adapter also stores which pool the session belongs to.
- `MFA_SETUP` on the admin pool: the adapter associates a software token and returns the secret and an `otpauth://` URI (issuer "OpEx", account the email). The UI renders the QR code client-side and shows the secret as text; neither is sent anywhere but Cognito. The first code verifies the token and completes the challenge.
- `accessToken()` returns Amplify's current access token, which Amplify refreshes when it has expired; `forceRefresh` forces a refresh-token exchange.
- `signOut()` revokes the refresh token with Cognito, then clears local tokens even if revocation fails (offline).

### LOCAL adapter

- Present only in bundles whose `ProfileConfig.auth.kind` is `local`; it is loaded through a dynamic import that the build eliminates for other profiles.
- `signIn` is replaced by choosing one of the configured development identities on the LOCAL sign-in page; the identity's token becomes the session's access token. There is no password, challenge, refresh, change, or reset; those UI actions are hidden in LOCAL.
- An expired development token produces a `401` from the local API, which ends the session and returns to the LOCAL sign-in page with "This development token has expired; mint a new one with `dotnet user-jwts`."

## User Context

```ts
type UserContext = {
  pool: Pool;
  userId: string;           // the access token's `sub`; stable across email changes
  email: string;            // display only: from the ID token; the identity's label in LOCAL
  role: 'opex-editor' | 'opex-admin' | null;
  tenants: { kind: 'all' } | { kind: 'listed'; tenants: TenantId[] };
};
```

One claims adapter builds it. It decodes the JWT payloads without verifying them.

| Session from | `role` | `tenants` |
|---|---|---|
| Main pool | `opex-editor` if the access token's `cognito:groups` contains it; else `null` | `listed`: every `tenant-{tenantId}` group whose id is 24 lowercase hexadecimal characters, in group order, de-duplicated; other `tenant-` groups are ignored |
| Admin pool | `opex-admin` | `all`, whatever tenant groups the token holds |
| LOCAL identity | Same rules, applied to the development token's `cognito:groups` claim, with the pool taken from the identity's configured `pool` | Same rules |

- A main-pool user with `role: null` or no tenant groups is signed in but has nothing to see; the shell shows "Your account has no access yet" with a link to request access, and sends no tenant-scoped requests.
- The administrator's tenant list comes from the API (see App Shell), not from claims.

## Capabilities

Areas decide what to render through one pure function in `platform/auth`, so the HLD's capability table is encoded once:

```ts
can(user: UserContext | 'demo', capability: Capability, tenant: TenantId): boolean
```

| Capability | demo | editor | admin |
|---|---|---|---|
| `viewReporting`, `viewInsights` | demo tenant only | listed tenants | any |
| `viewImprovement` | — | listed tenants | any |
| `editTenantConfiguration`, `requestInsightsRun`, `giveAnomalyFeedback`, `writeImprovementTickets` | — | listed tenants | any |
| `readAccessRequests`, `manageTenants` (list and create tenants) | — | — | ✓ |
| `submitAccessRequest` | ✓ | ✓ | ✓ |

A `can` that returns `false` hides or disables the control; it never replaces the API's authorization.

## Tab Handoff and Sign-out Across Tabs

All tabs share one same-origin `BroadcastChannel` named `opex-session`.

```
new tab, no tokens ── {type:'session-request', nonce} ──▶ channel
signed-in tab      ◀────────────────────────────────────
signed-in tab      ── {type:'session-offer', nonce, pool, entries} ──▶ new tab
new tab: first offer for its nonce within 300 ms wins → write entries to sessionStorage → restore()
no offer within 300 ms → state follows the route (none or demo)

any tab signs out ── {type:'signed-out'} ──▶ every tab clears tokens → landing page
```

- Only tabs in `signedIn` answer requests. A tab that receives an offer it did not request (wrong nonce) ignores it.
- `entries` are the session's `sessionStorage` key-value pairs (tokens and pool marker). The channel is same-origin by definition; nothing leaves the browser.
- The shell waits for handoff to settle before routing, showing a neutral loading state, so a new tab does not flash the landing page.
- Signing in, in one tab, does not sign in tabs that are already open in `none` or `demo`.

## Session Expiry and API Rejection

- `sessionRejected()` (the API returned 401 after a forced refresh) and a failed refresh both end the session: tokens are cleared in this tab, and the user is sent to the sign-in route of the session's pool with "Your session has ended. Sign in again." and a return-to path.
- `demoRejected()` (the API answered a demo read with 401, meaning the tenant is no longer anonymously readable) clears the API Client's cached demo-tenant list and sends the visitor to `/demo` with "That demo is no longer available". The visitor stays in the demo state; `/demo` offers "Sign in" for members.
- The return-to path is accepted only if it is a same-origin relative path beginning with `/` and not `//`; otherwise the user lands on their home.
- Ending a session this way keeps form drafts so the user can resume after signing in again (see Forms). Signing out, in any tab, discards them along with the tokens.
- A `403` never ends the session. The calling view shows that the action is not permitted for this account (see App Shell).

## Sign-in Pages

| Route | Contents |
|---|---|
| `/sign-in` | Email, password, "Forgot password", link to request access. Followed by a new-password step when required. |
| `/admin/sign-in` | Email, password, "Forgot password". Followed by a new-password step, and then a TOTP setup step (QR + secret + first code) or a TOTP code step. |
| `/sign-in/forgot`, `/admin/sign-in/forgot` | Email → emailed code + new password |
| `/account/password` | Signed-in only, Cognito pools only: current and new password |

- Opening a sign-in route while signed in shows "You're signed in as {email}" with "Go to OpEx" and "Sign out" instead of the form, including when the route is for the other pool.
- Password inputs allow paste and password managers (`autocomplete` attributes set for current/new password and one-time code).
- After `done`, the user goes to the return-to path, or else their home.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Multi-pool with Amplify | Reconfigure Amplify for the route's pool before sign-in and for the stored pool before restore | Two Amplify instances; `amazon-cognito-identity-js` | Amplify is a singleton, and a tab only ever has one session, so one configuration at a time is sufficient; the other library is in maintenance mode. |
| Demo/none state storage | Derived from the route | A stored mode flag | Nothing to clear or keep consistent; a shared demo link works in any fresh tab. |
| Signed-in tab on a demo route | Redirect to signed-in home | Show demo alongside the session | One state per tab keeps the request guard and the demo containment metric simple to reason about. |
| Handoff timeout | 300 ms, first offer wins | Longer wait; leader election | Same-machine channel replies are near-instant; a short wait keeps cold starts fast. |
| Uniform sign-in error | One message for unknown user, wrong password, disabled user | Distinct messages | Prevents account enumeration from the sign-in form. |
| Capability policy location | Pure `can()` in `platform/auth` | Per-area role checks; in the app shell | Areas cannot import the shell, and one encoding of the capability table keeps the areas consistent with each other and with the HLD. |

## Open Questions & Future Decisions

### Deferred
1. Token lifetimes for each pool's app client, and whether Cognito refresh-token rotation is enabled — rotation affects tabs that share a refresh token through handoff.
2. Whether an idle timeout (signing out after inactivity) is wanted beyond token lifetimes.
3. The claim shape `opex-web-api`'s local issuers accept for development tokens; this design assumes they carry `cognito:groups`.

## References

- `docs/high-level-design.md` — Sessions; Sign-in; Roles and tenants; Key Design Decisions 2–5
- `docs/intent/api-client/api-client-design.md` — `SessionCredentials` port
- `docs/intent/profile-config/profile-config-design.md` — pool configuration and LOCAL identities
- `docs/intent/app-shell/app-shell-design.md` — routing, home, not-permitted presentation
