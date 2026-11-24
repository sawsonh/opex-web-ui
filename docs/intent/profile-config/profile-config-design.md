---
parent: high-level-design
prefix: CFG
---

# Profile Config

## Context and Design Philosophy

The UI runs in three profiles — `LOCAL`, `QA`, and `PROD` — that differ only in configuration: which sign-in adapter is used, and the Cognito region and pool identifiers. Profile Config is the single place those values enter the application. Every other module receives a typed, already-validated `ProfileConfig`; none reads environment variables, and none branches on the profile name.

Three principles govern it:

- **Fixed at build time.** Each profile's bundle is built with that profile's values (HLD decision 11). There is no runtime configuration fetch.
- **Fail before shipping, fail closed at runtime.** Invalid or missing values fail the build. If a bundle somehow starts with invalid values, the app shows a configuration error and makes no API or Cognito calls.
- **Nothing secret.** Every value ends up in a public bundle. Secrets never pass through this module; LOCAL development tokens are the one exception allowed in a bundle, and only in the LOCAL profile, which is never deployed.

## Configuration Values

Values arrive as Vite build-time environment variables (prefix `VITE_`). In CI they come from GitHub Environment variables; in `LOCAL` from `.env.local` files that are git-ignored.

| Variable | Profiles | Meaning | Validation |
|---|---|---|---|
| `VITE_PROFILE` | all | `LOCAL`, `QA`, or `PROD` | One of the three, exactly |
| `VITE_COGNITO_REGION` | QA, PROD | AWS region of both pools | AWS region format (e.g. `eu-west-1`) |
| `VITE_MAIN_USER_POOL_ID` | QA, PROD | Main pool id | Cognito user pool id format (`{region}_{id}`), region prefix equals `VITE_COGNITO_REGION` |
| `VITE_MAIN_USER_POOL_CLIENT_ID` | QA, PROD | Main pool app client id | Non-empty alphanumeric |
| `VITE_ADMIN_USER_POOL_ID` | QA, PROD | Admin pool id | As main pool; must differ from the main pool id |
| `VITE_ADMIN_USER_POOL_CLIENT_ID` | QA, PROD | Admin pool app client id | Non-empty alphanumeric |
| `VITE_LOCAL_IDENTITIES` | LOCAL | Development identities for the LOCAL sign-in page | JSON array, see below |

Cognito variables are rejected in a `LOCAL` build, and `VITE_LOCAL_IDENTITIES` is rejected in `QA` and `PROD` builds, so a misconfigured pipeline cannot ship a development identity or point LOCAL at a real pool by accident.

### LOCAL development identities

`VITE_LOCAL_IDENTITIES` is a JSON array of `{ "label": string, "pool": "main" | "admin", "token": string }`. Each `token` is an access token minted with `dotnet user-jwts` against `opex-web-api`'s local issuer for that pool. Labels are shown on the LOCAL sign-in page (e.g. "Editor — tenant acme"). The array must be non-empty, labels unique, and each token a syntactically well-formed JWT (three base64url segments). Token expiry is not checked here; an expired token surfaces as a `401` from the local API.

## The `ProfileConfig` Type

```ts
type ProfileConfig = {
  profile: 'LOCAL' | 'QA' | 'PROD';
  auth:
    | { kind: 'local'; identities: LocalIdentity[] }
    | { kind: 'cognito'; region: string; mainPool: PoolConfig; adminPool: PoolConfig };
};
type PoolConfig = { userPoolId: string; clientId: string };
```

The sign-in adapter is chosen from `auth.kind`, not from `profile`. `profile` exists for display (an environment badge outside `PROD`) and diagnostics only; no code branches on it for behavior.

## Loading and Validation

```
vite.config.ts ── validateProfileEnv(env) ──▶ build fails with every problem listed
                                │
main.tsx ── loadProfileConfig(import.meta.env) ──▶ ProfileConfig | ConfigError
                                │
                    app composition root injects ProfileConfig
                    into platform/auth, platform/api, and the shell
```

- `validateProfileEnv` and `loadProfileConfig` share one pure parser in `platform/config` with no dependency on Vite, React, or the browser. The parser reports **all** problems at once, naming each variable and why it is invalid, never echoing a value.
- The build calls it from the Vite config, so the Build stage fails before bundling.
- At startup the composition root parses `import.meta.env` once. On success it passes the frozen `ProfileConfig` to the modules that need it by constructor or provider injection. On failure it renders a static "This deployment is misconfigured" page and starts nothing else.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Where validation runs | Same parser at build time (Vite config) and at startup | Build time only; startup only | Build-time validation stops a bad bundle reaching S3; the startup check is a cheap second line when a bundle is built outside the pipeline. |
| How modules get config | Injected from the composition root | A global `getConfig()` singleton | Injection lets tests and stories pass any config without module mocking, and keeps `platform/*` free of startup ordering assumptions. |
| Adapter selection key | `auth.kind` discriminant | Branching on `profile` | Behavior follows the shape of the values, so a new profile needs no code changes and no inline `if (profile === …)`. |
| Cross-profile rejection | Cognito variables forbidden in LOCAL, LOCAL identities forbidden in QA/PROD | Ignore extra variables | A stray development token in a deployed bundle would be a credential leak; rejecting it costs nothing. |
| LOCAL identities as config | JSON in a git-ignored `.env.local` | A checked-in fixture file; fetching tokens from the local API at runtime | Tokens are developer-specific and expire; keeping them out of git and out of runtime calls keeps LOCAL simple and leak-free. |

## Open Questions & Future Decisions

### Deferred
1. Feature flags: none are needed by the current HLD. When one is, it is added here as a typed, validated value rather than read ad hoc.

## References

- `docs/high-level-design.md` — Environment profiles; Key Design Decision 11
- `docs/intent/auth-session/auth-session-design.md` — consumes `auth`
- `docs/intent/delivery-pipeline/` — sets the variables per GitHub Environment
