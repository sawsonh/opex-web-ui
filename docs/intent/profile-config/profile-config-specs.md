# Profile Config — EARS Specs

Design: `profile-config-design.md`

## Parsing and validation

- [ ] **CFG-001**: The profile configuration parser shall accept `VITE_PROFILE` only when it is exactly `LOCAL`, `QA`, or `PROD`.
- [ ] **CFG-002**: When `VITE_PROFILE` is `QA` or `PROD`, the profile configuration parser shall require `VITE_COGNITO_REGION`, `VITE_MAIN_USER_POOL_ID`, `VITE_MAIN_USER_POOL_CLIENT_ID`, `VITE_ADMIN_USER_POOL_ID`, and `VITE_ADMIN_USER_POOL_CLIENT_ID`.
- [ ] **CFG-003**: When `VITE_PROFILE` is `QA` or `PROD`, the profile configuration parser shall reject a Cognito region that is not in AWS region format, a user pool id that is not in `{region}_{id}` format or whose region prefix differs from `VITE_COGNITO_REGION`, an app client id that is empty or not alphanumeric, and identical main and admin user pool ids.
- [ ] **CFG-004**: When `VITE_PROFILE` is `QA` or `PROD`, the profile configuration parser shall reject the configuration if `VITE_LOCAL_IDENTITIES` is set.
- [ ] **CFG-005**: When `VITE_PROFILE` is `LOCAL`, the profile configuration parser shall reject the configuration if any of the Cognito variables (`VITE_COGNITO_REGION`, `VITE_MAIN_USER_POOL_ID`, `VITE_MAIN_USER_POOL_CLIENT_ID`, `VITE_ADMIN_USER_POOL_ID`, `VITE_ADMIN_USER_POOL_CLIENT_ID`) is set.
- [ ] **CFG-006**: When `VITE_PROFILE` is `LOCAL`, the profile configuration parser shall require `VITE_LOCAL_IDENTITIES` to be a non-empty JSON array of `{ label, pool, token }` entries with unique labels, `pool` equal to `main` or `admin`, and each `token` a syntactically well-formed JWT (three base64url segments).
- [ ] **CFG-007**: If the profile configuration is invalid, then the profile configuration parser shall report every problem found, each naming the variable and the reason, without including any variable's value.
- [ ] **CFG-008**: The profile configuration parser shall have no dependency on Vite, React, or browser APIs.

## Build and startup

- [ ] **CFG-009**: When the bundle is built, the Vite configuration shall run the profile configuration parser on the build environment and fail the build if the configuration is invalid.
- [ ] **CFG-010**: When the application starts, the composition root shall parse the profile configuration once and pass the resulting frozen `ProfileConfig` to the modules that need it.
- [ ] **CFG-011**: If the profile configuration is invalid at startup, then the application shall render a static "This deployment is misconfigured" page and make no API or Cognito requests.

## Shape and use

- [ ] **CFG-012**: The `ProfileConfig` shall express the sign-in adapter as an `auth` value discriminated by `kind`: `local` with the development identities, or `cognito` with the region and both pools' user pool id and app client id.
- [ ] **CFG-013**: The application shall select its sign-in adapter from `ProfileConfig.auth.kind`, and no module shall branch on `ProfileConfig.profile` for behavior other than display.
