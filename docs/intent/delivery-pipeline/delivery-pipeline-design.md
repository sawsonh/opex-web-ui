---
parent: high-level-design
prefix: PIPE
---

# Delivery Pipeline

## Context and Design Philosophy

Every pull request is built, tested, deployed to QA, and smoke-tested there; every commit to `main` goes through the same stages to PROD. The pipeline is GitHub Actions, and each stage gates the next: Build → Test → Deploy → Smoke Test.

Principles:

- **One pipeline, two targets.** QA and PROD run the same reusable workflow with a different GitHub Environment, so what is verified in QA is what runs in PROD.
- **The bundle that was tested is the bundle that ships.** Build produces one artifact per run; Deploy publishes that artifact and nothing else.
- **No standing credentials.** AWS access is short-lived through GitHub OIDC; nothing secret is stored for Build, Test, or Smoke Test.
- **Deploys never break open tabs.** Assets from the previous release stay available after a deploy, so a user mid-session can still load lazily loaded chunks.

## Workflows

| File | Trigger | Environment | Concurrency |
|---|---|---|---|
| `.github/workflows/pipeline.yml` | `workflow_call` with input `environment` | — (reusable) | — |
| `.github/workflows/qa.yml` | `pull_request` to `main` | `qa` | group `qa`, `cancel-in-progress: true` |
| `.github/workflows/prod.yml` | `push` to `main` | `prod` | group `prod`, `cancel-in-progress: false` |

- Default workflow permissions are `contents: read`. Only the Deploy job adds `id-token: write`.
- Third-party actions are pinned to a full commit SHA.
- The Node.js version comes from `.nvmrc`; dependencies install with `npm ci`, using the npm cache keyed on `package-lock.json`.

## Stages

### Build

1. `npm ci`
2. `npm run api:generate` — types from the committed OAS snapshot
3. `npm run typecheck` — `tsc --noEmit`
4. `npm run lint` — ESLint with `--max-warnings 0`, including the import-boundary rules
5. `npm run format:check` — Prettier
6. `npm run build` — `vite build`, with the profile's `VITE_*` variables from the GitHub Environment; Profile Config's validation fails the build on any invalid value
7. **Bundle guard**: the build output is searched for the LOCAL sign-in adapter's marker string and for `VITE_LOCAL_IDENTITIES` content; any match fails the build
8. Upload `dist/` as the run's bundle artifact

### Test

1. `npm run test:coverage` — Vitest with V8 coverage; line, branch, function, and statement thresholds of 90% each, configured in `vitest.config.ts`. Below threshold fails the stage.
2. Upload the coverage report as an artifact; post the summary to the job summary.

### Deploy

Skipped for pull requests from forks (no OIDC token): the job runs only when the pull request's head repository is this repository, or on `push` to `main`.

1. Assume `vars.DEPLOY_ROLE_ARN` in `vars.AWS_REGION` via `aws-actions/configure-aws-credentials` (OIDC).
2. Download the run's bundle artifact.
3. Upload hashed assets (`assets/**`) with `Cache-Control: public, max-age=31536000, immutable`, without deleting anything.
4. Upload `index.html` and other unhashed files with `Cache-Control: no-cache`.
5. Write a release manifest listing this release's files to `.releases/{commit-sha}.json` in the bucket, and record it as current.
6. Delete hashed assets that are in neither the current nor the previous release's manifest.
7. Create a CloudFront invalidation for `/index.html` and `/`, and wait for it to complete.

### Smoke Test

Playwright against `vars.APP_URL`, with no credentials:

1. The landing page loads and shows "View demo".
2. "View demo" opens a reporting dimension page, choosing the first demo tenant if a picker appears.
3. At least one `/v1/tenants/…` request completes with `200`, and no request carried an `Authorization` header.
4. No uncaught errors in the browser console.

Each check retries for up to 2 minutes to absorb CloudFront propagation. On failure, the Playwright trace is uploaded as an artifact and the workflow fails.

## Environment Values

Set on the `qa` and `prod` GitHub Environments to the values `opex-infra-cdk` outputs. All are variables, not secrets.

| Variable | Used by |
|---|---|
| `AWS_REGION`, `DEPLOY_ROLE_ARN`, `S3_BUCKET`, `CLOUDFRONT_DISTRIBUTION_ID` | Deploy |
| `APP_URL` | Smoke Test |
| `COGNITO_REGION`, `MAIN_USER_POOL_ID`, `MAIN_USER_POOL_CLIENT_ID`, `ADMIN_USER_POOL_ID`, `ADMIN_USER_POOL_CLIENT_ID` | Build (as `VITE_*`) |

Fork pull requests build with the `qa` variables, which are not secret, and stop after Test.

## Rollback

A bad release is rolled back by reverting its pull request; the revert's merge to `main` runs the pipeline and deploys the previous code. Because the previous release's assets are kept, a re-run of an earlier `prod` workflow is also safe.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Workflow structure | One reusable workflow called by QA and PROD workflows | Two independent workflows | QA and PROD cannot drift in their stages or checks. |
| Asset retention | Keep current and previous release's hashed assets; prune the rest | `aws s3 sync --delete`; never delete; S3 lifecycle expiry | `--delete` breaks open tabs loading lazy chunks; never deleting grows without bound; age-based expiry can delete live assets that are unchanged across releases. |
| Invalidation scope | `index.html` and `/` only | `/*` | Hashed assets are immutable by name; invalidating everything costs time and evicts cached assets for nothing. |
| Smoke-test identity | Demo only, no credentials | A named test user | Needs no secrets and cannot change data or notify the administrator (HLD); a sign-in check remains an HLD open question. |
| Bundle guard | Search build output for LOCAL markers | Trust build-time elimination | Dead-code elimination is easy to break silently; a grep makes it a failing check. |

## Open Questions & Future Decisions

### Deferred
1. Whether deployed environments also get an automated sign-in check (HLD open question).
2. Whether merging requires the QA workflow to pass (branch protection is a repository setting, outside this repo's files).

## References

- `docs/high-level-design.md` — Delivery pipeline; Key Design Decisions 11–13
- `docs/intent/profile-config/profile-config-design.md` — build-time variables and validation
- `opex-infra-cdk` — bucket, distribution, deploy role, and their outputs
