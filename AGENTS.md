# AGENTS.md — opex-web-ui

Instructions for AI coding agents (and humans) working in this repository.

> **All work in this repo follows Spec-Driven Development using the `linked-intent-dev` (LID) skill. This is not optional.** See [Linked-Intent Development](#linked-intent-development-mandatory) below. No code, test, config, or infrastructure-facing change lands without walking the arrow of intent.

## Project overview

`opex-web-ui` is the frontend for **Operational Excellence (OpEx) reporting**: a single-page application where anyone can explore a demo tenant without signing in; editors examine their tenants' performance across engineering health dimensions, read insights and give feedback on flagged anomalies, drive improvement through DMAIC with RCA and action tickets, configure which source-system items are collected for their tenants, and request insights runs for them; and the administrator reads access requests and acts on any tenant.

### The OpEx system

| Repo | Responsibility |
|---|---|
| `opex-collector-worker` | Collects data from external sources (SonarQube, GitHub, Jira, PagerDuty, …), aggregates it, and shapes it into an analytical model stored in MongoDB Atlas. |
| `opex-insights-workflow` | Produces a tenant's report for a date range: per section, an executive summary, health score, anomalies, RCA assistance, and recommended actions. Runs are requested through `opex-web-api` by an editor for one of their tenants or by the administrator for any tenant. |
| `opex-web-api` | The API this UI reads from and writes to. Publishes an OpenAPI Specification (OAS) that is the contract between API and UI, and persists tenant configuration and user-created records in MongoDB Atlas. |
| `opex-infra-cdk` | AWS CDK app that provisions the infrastructure (including this UI's S3 bucket and CloudFront distribution). |
| `opex-web-ui` (this repo) | React SPA for exploring metrics and insights, running DMAIC improvement work, configuring tenants, and submitting access requests. |

This UI never talks to source systems or MongoDB directly — **every read and every write goes through `opex-web-api`**. The UI never edits the analytical model or insights; its writes are tenant configuration, anomaly feedback, improvement tickets, access requests, and insights run requests.

### Domain: reporting dimensions and metrics

Each reporting dimension presents data shaped by the collector from a specific origin system. The list is not exhaustive; more dimensions and metrics will be added.

| Dimension | Origin | Example metrics |
|---|---|---|
| Code Quality | SonarQube | # of Bugs / Code Smells / Vulnerabilities, Duplication Density, Code Coverage |
| Test Coverage | GitHub | Coverage trends |
| Development Velocity | GitHub | PR Cycle Time (Ready for Review → Merge), PR Complexity, PR Complexity vs. Comment ratio, PR Time to First Engagement, PR Time from First Engagement → Merge |
| Work Throughput | Jira | # of Tickets Closed, WIP Cycle Time |
| Support Health | PagerDuty | MTTA (Mean Time to Acknowledge), MTTR (Mean Time to Resolve) |

### Domain: beyond reporting

| Area | Who | What it covers |
|---|---|---|
| Insights | Analysts; editors and the administrator request runs | Per-section **insights** in each **report** (one per insights run over a date range), **anomaly feedback** on flagged anomalies (in progress), **insights run** requests, and the **paused** state set by the budget guard |
| Improvement | Analysts | **DMAIC** (Define, Measure, Analyze, Improve, Control): **RCA tickets** and **action tickets** raised against a bad metric, carried through to verified improvement |
| Access Requests | Anyone; the administrator | The public **access request** form and the administrator's list of requests |
| Tenant Administration | Editors; the administrator | **Tenant configuration** of which GitHub repos, SonarQube projects, Jira projects, PagerDuty services, and other source items are collected; the administrator also creates **tenants**, each with a **tenant type** (`DEMO`, `CUSTOMER`, `SANDBOX`) |

Use these names — dimension, metric, origin, tenant, tenant type, date range, report, insights, insights run, health score, anomaly, anomaly feedback, DMAIC, RCA ticket, action ticket, tenant configuration, access request, demo tenant — consistently in code, specs, and docs (ubiquitous language). A **tenant** is one engineering team; there is no separate team concept within a tenant. Model new dimensions and areas as new bounded areas rather than special cases inside existing ones.

## Tech stack & hosting

- **React + TypeScript** (strict mode) single-page application.
- **Plotly.js** for charts and visualizations.
- Built as static assets, **hosted on S3, delivered globally via CloudFront**.
- Infrastructure is owned by `opex-infra-cdk`; do not hand-provision AWS resources from this repo.

## Architecture principles (required)

- **Clean Architecture** — dependencies point inward. Domain logic (metric definitions, calculations, formatting rules, value objects) has no dependency on React, Plotly, HTTP, or browser APIs. UI components and API adapters live at the edges and depend on the domain, never the reverse.
- **Domain-Driven Design** — organize by domain (e.g. Code Quality, Development Velocity, Work Throughput, Support Health) rather than by technical layer alone. Keep the ubiquitous language above in names.
- **SOLID / DRY** — small, single-purpose units; depend on abstractions (ports) for API access, auth, and configuration so they can be substituted in tests and per profile. Extract shared chart, metric, and layout behavior instead of duplicating it per dimension.

Concrete layering, folder structure, and library choices are recorded in `docs/high-level-design.md` and the LLDs under `docs/intent/`, not here.

## Environment profiles

| Profile | Runs where | AWS dependencies |
|---|---|---|
| `LOCAL` | Developer machine | Emulated by **ministack** running as a local Docker container |
| `QA` | AWS | Real AWS |
| `PROD` | AWS | Real AWS |

- The API base URL is the relative path `/v1` in every profile (CloudFront serves the SPA and the API on one origin; the `LOCAL` dev server proxies `/v1`). All other environment-specific values (Cognito pool and client IDs, region, feature flags, etc.) come from profile configuration — never hard-coded and never branched on inline throughout the code.
- `LOCAL` must be runnable end-to-end without access to AWS.
- Never commit secrets or credentials for any profile.

## API integration & auth

- **The `opex-web-api` OAS is the source of truth for the API contract.** Prefer generating TypeScript types / client code from the OAS over hand-writing request/response shapes; keep any hand-written mapping in an adapter layer that translates API DTOs into domain types.
- **Demo without signing in.** "View demo" opens a demo tenant's reporting dimensions and insights, read-only — no anomaly feedback, no insights run requests, no edits. `opex-web-api` lists the demo tenants — tenants whose type is `DEMO` — at `GET /v1/demo-tenants`, anonymously; the UI never hard-codes or configures a demo tenant id. In the demo state the UI sends only that listing and `GET` reporting and insights reads under `/v1/tenants/{tenantId}/…` for a listed demo tenant, with no `Authorization` header. A `401` in the demo state means the tenant is no longer public: the UI refreshes the demo-tenant list and returns the visitor to the demo picker.
- **Named users sign in with the SPA's own forms against Amazon Cognito (SRP)** — no hosted UI, no callback URLs, no self sign-up. Two pools: **main** (`opex-editor`; email + password) at `/sign-in`, and **admin** (`opex-admin`; email + password + authenticator-app MFA) at `/admin/sign-in`. In `LOCAL`, development identities minted with `dotnet user-jwts` stand in, behind the same auth port. Cognito issues and signs the JWTs; the UI never creates or verifies tokens.
- **Send the Cognito access token** (never the ID token) as the `Bearer` token on every `/v1` call except demo reads and the public access-request submission.
- **Roles and tenants:** a main-pool token's role is `opex-editor` when it holds that group, and its tenants are its `tenant-{tenantId}` groups; an admin-pool token is `opex-admin` for every tenant, whatever tenant groups it holds. Only the claims adapter in `platform/auth` reads Cognito claim names. The UI uses role and tenants only to show or hide things; `opex-web-api` enforces them.
- **Tenant-scoped URIs.** API routes include the active tenant id in the URI, and the API validates it against the token. The UI only ever addresses a tenant the user belongs to, and must handle authorization failures (401/403) gracefully — token expiry, re-authentication, insufficient role, and tenant mismatch.
- Treat tokens as sensitive: no logging, no exposure in URLs, and storage choices documented in the relevant LLD.

## Git workflow

- **No direct commits to `main`.** Every change reaches `main` through a pull request.
- **Pull requests are squash-merged** into `main`, so each PR becomes a single commit on `main`.
- **Branch names** follow `feature/{short-description}`, where the description is a short kebab-case summary of the change — e.g. `feature/pr-cycle-time-chart`, `feature/tenant-scoped-api-client`.
- Agents: before committing, create (or switch to) a correctly named `feature/…` branch; never commit or push to `main`.

## CI/CD

**GitHub Actions** is the CI/CD pipeline. Every workflow runs four stages in order, and each stage gates the next:

```
Build → Test → Deploy → Smoke Test
```

| Trigger | Deploy target | Smoke-tested against |
|---|---|---|
| Pull request commits | `QA` | `QA` |
| Commits to `main` (squash merges of approved PRs) | `PROD` | `PROD` |

- **Build** — install, type-check, lint, format check, and produce the static bundle for the target profile.
- **Test** — unit/component tests; fails if the coverage threshold is not met.
- **Deploy** — publish the bundle to the target profile's S3 bucket and invalidate its CloudFront distribution.
- **Smoke Test** — verify the deployed app in the target environment loads and can reach `opex-web-api`; a failure fails the workflow.
- **AWS access uses GitHub OIDC.** The Deploy stage assumes the environment's IAM role through GitHub's OIDC provider. No long-lived AWS keys are stored in GitHub. The role and its trust policy are provisioned by `opex-infra-cdk`.
- **Fork PRs** get no OIDC token, so they run Build and Test only and skip Deploy and Smoke Test.
- Per-environment values (role ARN, region, S3 bucket, CloudFront distribution ID) are stored as GitHub Environment variables (`qa`, `prod`), set to match what `opex-infra-cdk` provisions. Never hard-code them in workflows. Smoke tests use the demo and need no credentials.

## Quality gates

- **Testing** — a high level of test coverage is required. Tests are written *before* the code that satisfies them (LID Phase 5). Domain logic gets thorough unit tests; components and adapters get behavior-focused tests with the API mocked at the boundary. The coverage threshold is defined in the HLD and enforced in CI.
- **Linting & formatting** — the project uses proper linting for TypeScript and React (type-aware lint rules, React Hooks rules, accessibility rules) plus an automatic formatter. Lint and format checks run locally and in CI; do not disable rules inline without a justifying comment.
- **Type safety** — TypeScript `strict` is on. Avoid `any`; narrow unknown API data at the adapter boundary.

Before considering any change complete: type-check, lint, format check, and the full test suite (with coverage) must pass, plus LID coherence verification.

---

## LID
- Mode: Full
- Version: 1.3.0

## Linked-Intent Development (MANDATORY)

**Consult the `linked-intent-dev` skill for ALL code changes.** All changes flow through the arrow of intent in one direction:

```
HLD → LLDs → EARS → Tests → Code
```

- **New features and refactors**: full six-phase workflow (HLD check → LLD check/draft → EARS → intent-narrowing edge audit → tests-first → code).
- **Bug fixes**: walk the arrow like any other change — find where behavior diverged from intent and cascade from there. No short-circuit.
- **If unsure**: use the full workflow.

Stop after each phase for user review. **Docs carry current intent, written to be read cold** — write each doc as if authored fresh today, from current intent alone: no narration of how it changed, no meaning that needs the conversation that produced it, no rebuttals to questions only a past discussion raised. Rationale, considered alternatives, and constraints a fresh author would independently write stay; record rejected alternatives and why in the LLD's Decisions & Alternatives table, not as asides in body prose.

**Memory vs. intent.** Before saving durable project knowledge to agent or tool memory, test whether it is project *intent* — would a fresh agent, in any tool, next session, need it to build this system correctly? If yes, record it in the arrow (HLD / LLD / EARS / decision doc), which travels and cascades — not in private, per-tool memory, where intent escapes the arrow. Knowledge about the user or how they like to work stays in memory.

### Navigation

| What you need | Where to look |
|---|---|
| High-level design | `docs/high-level-design.md` |
| Design tree (sub-HLDs, LLDs, their specs) | `docs/intent/` — one folder per node |
| EARS specs | beside each design doc as `{node}-specs.md` in the node's folder under `docs/intent/` |
| Decision docs | `docs/decisions/` (project-level) and `docs/intent/<segment>/decisions/` |

### Terminology

- **HLD**: High-Level Design — single project-level doc at `docs/high-level-design.md`.
- **LLD**: Low-Level Design — detailed component design doc in `docs/intent/`. The design layer is a recursive tree: the root is the HLD, leaf LLDs own EARS, and a component deep enough to outgrow one doc becomes a sub-HLD (HLD-shaped, owns no EARS) with children beneath it. "HLD" and "LLD" are roles by position; depth-2 (one HLD over flat leaf LLDs) is the default.
- **EARS**: Easy Approach to Requirements Syntax — structured one-line requirements beside each design doc as `{node}-specs.md` in the node's folder under `docs/intent/`. IDs are path-concatenated — the root-to-leaf path of the owning segment plus a number — so a prefix grep gathers a subtree. Markers: `[x]` implemented, `[ ]` active gap, `[D]` deferred.
- **Arrow**: the unidirectional chain from vision to code (HLD → LLDs → EARS → Tests → Code). Strictly a DAG of intent.
- **Arrow segment**: the territory owned by one leaf LLD — the LLD itself plus the specs, tests, and code that cite its EARS IDs. The boundary is the leaf prefix. Within-segment cascade is free; across-segment cascade pauses.
- **Cascade**: propagating a change downstream through the arrow so adjacent levels stay coherent.

### Code annotations

Annotate code and tests with `@spec` comments citing EARS IDs:

```ts
// @spec AUTH-UI-001, AUTH-UI-002
```

Place the annotation at the *entry point of the behavior's implementation graph* — the topmost function or module owning the specified behavior, not every helper. When a behavior spans multiple subsystems (UI + API + database, for example), annotate at the entry point in each subsystem. Tests follow the same rule: annotate the test that directly exercises the spec, not every inner assertion.
