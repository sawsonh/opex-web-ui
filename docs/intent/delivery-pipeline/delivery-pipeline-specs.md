# Delivery Pipeline — EARS Specs

Design: `delivery-pipeline-design.md`

## Workflows

- [ ] **PIPE-WF-001**: The QA workflow (`qa.yml`, on pull requests to `main`) and the PROD workflow (`prod.yml`, on pushes to `main`) shall each run the reusable workflow `pipeline.yml` with their GitHub Environment (`qa` or `prod`).
- [ ] **PIPE-WF-002**: The QA workflow shall use concurrency group `qa` with `cancel-in-progress: true`; the PROD workflow shall use concurrency group `prod` with `cancel-in-progress: false`.
- [ ] **PIPE-WF-003**: The pipeline shall run Build, Test, Deploy, and Smoke Test in that order, each starting only if the previous one succeeded.
- [ ] **PIPE-WF-004**: The pipeline's default permissions shall be `contents: read`, with `id-token: write` granted only to the Deploy job.
- [ ] **PIPE-WF-005**: The pipeline shall pin every third-party action to a full commit SHA, take the Node.js version from `.nvmrc`, and install dependencies with `npm ci`.

## Build

- [ ] **PIPE-BUILD-001**: The Build stage shall generate API types, type-check, lint with zero warnings allowed, check formatting, and build the bundle with the environment's `VITE_*` variables, failing on the first step that fails.
- [ ] **PIPE-BUILD-002**: If the built bundle contains the LOCAL sign-in adapter's marker string or any `VITE_LOCAL_IDENTITIES` content, then the Build stage shall fail.
- [ ] **PIPE-BUILD-003**: The Build stage shall upload the built bundle as the run's artifact, and the Deploy stage shall publish that artifact and nothing else.

## Test

- [ ] **PIPE-TEST-001**: The Test stage shall run the unit and component tests with V8 coverage and fail if line, branch, function, or statement coverage is below 90%.
- [ ] **PIPE-TEST-002**: The Test stage shall upload the coverage report as an artifact and write its summary to the job summary.

## Deploy

- [ ] **PIPE-DEPLOY-001**: When the workflow was triggered by a pull request from a fork, the pipeline shall skip Deploy and Smoke Test.
- [ ] **PIPE-DEPLOY-002**: The Deploy stage shall obtain AWS credentials only by assuming `vars.DEPLOY_ROLE_ARN` in `vars.AWS_REGION` through GitHub OIDC.
- [ ] **PIPE-DEPLOY-003**: The Deploy stage shall upload hashed assets with `Cache-Control: public, max-age=31536000, immutable` and unhashed files, including `index.html`, with `Cache-Control: no-cache`, uploading assets before `index.html`.
- [ ] **PIPE-DEPLOY-004**: The Deploy stage shall write a manifest of the release's files to `.releases/{commit-sha}.json` in the bucket and record it as the current release.
- [ ] **PIPE-DEPLOY-005**: The Deploy stage shall delete only hashed assets listed in neither the current nor the previous release's manifest.
- [ ] **PIPE-DEPLOY-006**: The Deploy stage shall invalidate `/index.html` and `/` in CloudFront and wait for the invalidation to complete.

## Smoke test

- [ ] **PIPE-SMOKE-001**: The Smoke Test stage shall open `vars.APP_URL` with Playwright, using no credentials, and verify that the landing page shows "View demo".
- [ ] **PIPE-SMOKE-002**: The Smoke Test stage shall choose "View demo", pick the first demo tenant if a picker appears, and verify that a reporting dimension page loads.
- [ ] **PIPE-SMOKE-003**: The Smoke Test stage shall verify that at least one `/v1/tenants/…` request returned `200`, that no request carried an `Authorization` header, and that the browser console has no uncaught errors.
- [ ] **PIPE-SMOKE-004**: The Smoke Test stage shall retry each check for up to 2 minutes before failing, and on failure upload the Playwright trace as an artifact.
