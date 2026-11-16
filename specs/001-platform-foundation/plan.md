# Implementation Plan: Platform Foundation

**Branch**: `4-spec-001-create-and-approve-spec-and-plan` | **Date**: 2026-11-16 | **Spec**:
[spec.md](./spec.md)

**Input**: Feature specification from `specs/001-platform-foundation/spec.md` (`ROADMAP.md` entry
001)

## Summary

Set up the platform that every later roadmap entry builds on:

- an npm-workspaces TypeScript monorepo with a React web app, Lambda backend and AWS CDK
  infrastructure;
- a GitHub Actions CI workflow whose jobs are the constitution's required checks (lint,
  type-check, unit tests with a 70% combined line-coverage gate, build, CDK synth) plus API tests
  on the Lambda Node version and a secret scan;
- a serverless AWS dev environment: CloudFront in front of S3 for the web page, an API Gateway
  HTTP API with Lambda for `/api/*`, RDS for PostgreSQL in isolated subnets, and a Cognito user
  pool;
- automatic deploys on every merge to `main` (and manual dispatch from `main`) through GitHub OIDC,
  with deploy access defined in a separate, manually deployed CDK app and a scoped CloudFormation
  execution policy with a permissions boundary instead of `AdministratorAccess`;
- a teardown workflow that leaves no application resources behind;
- structured logs with request IDs and 7-day retention, and a consistent error response.

The decisions and rejected alternatives are in [research.md](./research.md). They form the
architecture proposal that the full team must review before implementation (FR-013, Principle III).

## Technical Context

**Language/Version**: TypeScript 5.x (strict). Node.js from `.nvmrc` for tooling and CI; Lambda
runtime `nodejs24.x` (R7)

**Primary Dependencies**:

- Web: React 19 and Vite.
- API: AWS Lambda Powertools for TypeScript (Logger), `pg`, and the AWS SDK v3 Secrets Manager
  client.
- Infrastructure: AWS CDK v2 (`aws-cdk-lib`); Lambda code from esbuild output via
  `lambda.Code.fromAsset` (R12).
- Tooling: ESLint 9 with `typescript-eslint`, Prettier, Vitest, and `gitleaks` in CI.

**Storage**: Amazon RDS for PostgreSQL 17 (`db.t4g.micro`, single-AZ, isolated subnets). No
application tables in this entry (R2)

**Testing**: Vitest with workspace projects and `@vitest/coverage-v8` (lines ≥ 70% over all
application and infrastructure TypeScript source combined; `apps/api` tests also on Node 24),
React Testing Library with jsdom for the web, CDK `Template` assertions for infrastructure, and a
post-deploy smoke check (R6, R9)

**Target Platform**: AWS, one account and region for the shared dev environment; evergreen
desktop browsers for the web page

**Project Type**: Web application (SPA + serverless API) with infrastructure as code

**Performance Goals**: From the spec: required checks run in under 15 minutes (SC-004), a deploy
after merge finishes in under 20 minutes (SC-003), and a first deploy from a clean checkout takes
under 30 minutes (SC-002). No runtime throughput targets for a placeholder app

**Constraints**:

- HTTPS only (FR-006a).
- No NAT gateway and no long-lived AWS keys.
- Least-privilege IAM, including the CloudFormation execution role (scoped policy and permissions
  boundary, R4); deploys only from `main` (R3).
- Health check bounded: secret fetch, database connect and query each at 2 s (about 6 s worst
  case), Lambda timeout 10 s (R8).
- `CDK synth` runs without AWS credentials; required CI jobs are never skipped (R12).
- 7-day log retention.
- Destroy leaves no application resources retained (FR-012a); the ci-access app and the CDK
  bootstrap stack are shared account setup and are kept (R10).
- Deploys to the dev environment never run concurrently (FR-007).

**Scale/Scope**: One shared dev environment, a team of a few developers, and three workspaces;
a health endpoint and a placeholder page only

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

| Principle | Gate | Status |
| --- | --- | --- |
| I. Narrow, Explicit Scope | No Kanban features; nothing from the out-of-scope list | Pass: placeholder page, health check, user pool only |
| II. Spec-Driven Development | Approved spec; every module traces to an FR | Pass: spec clarified and approved by the user; formal approval is the spec-and-plan PR; mapping below |
| III. Team-Based Decisions | Architecture proposal reviewed by the full team before implementation | **Pending**: [research.md](./research.md) is the proposal; the spec-and-plan PR is its review venue. Implementation is blocked until every member has explicitly approved it |
| IV. Security-First | Documented security review before deploy; server-side authZ; no secrets in repo; least privilege | Pass by design: OIDC limited to `main` through the `dev` Environment, separate ci-access app, scoped CloudFormation execution policy and permissions boundary (R3, R4), scoped secret access, isolated DB with verified TLS, secret scanning. The ci-access app, execution policy and boundary are security-sensitive: a documented `security-auditor` review is required before their first deploy |
| V. Technology Baseline | TypeScript, React, RDS PostgreSQL, Cognito, AWS-managed, serverless preferred | Pass |
| VI. Real Deployment Target | Validated in AWS dev; CDK TypeScript; reproducible | Pass: deploy on merge, smoke check, destroy/redeploy in quickstart |
| VII. Quality Gate | Lint, type-check, unit tests, build, CDK synth; ≥70% coverage | Pass: one CI job per check, never skipped; API tests also on Node 24 (R7, R12) |
| VIII. Git Workflow | PRs only; required checks before merge | Pass: required checks are added to the `main` ruleset. Note: `required_approvals = 0` in `project-context.toml` conflicts with this principle's 1 approval and is outside this feature |
| IX. Simplicity and Operational Clarity | Simplest design; small increments; logging/error handling | Pass: no NAT, no RDS Proxy, no CDK Pipelines; Powertools logging |
| X. Traceable Work Tracking | Work tied to issues | Pass: issue #4 for this PR; Phase issues come from `tasks.md` |

Requirement traceability:

| Requirement | Where it is implemented |
| --- | --- |
| FR-001, FR-002, FR-003 | `.github/workflows/ci.yml`, `vitest.config.ts`, ruleset |
| FR-004 | `infra/` application app |
| FR-012a | `infra/` removal policies, `.github/workflows/destroy-dev.yml` |
| FR-005, FR-006, FR-006a | `infra/lib/*`, `apps/api/src/health`, `apps/web` |
| FR-007 | `.github/workflows/deploy-dev.yml`, `dev` Environment branch policy |
| FR-008, FR-009 | `infra/lib/data-stack.ts`, `infra/lib/api-stack.ts` (scoped secret grant), `infra/ci-access/`, `ci.yml` secret scan |
| FR-010, FR-011 | `apps/api/src/lib/`, `infra/lib/api-stack.ts` |
| FR-012 | `README.md` / `docs/development.md`, [quickstart.md](./quickstart.md) |
| FR-013 | This plan and [research.md](./research.md), reviewed in the spec-and-plan PR |

**Post-design re-check (after Phase 1, updated after code review)**: no new violations. The
contracts add no endpoints beyond `/api/health`, and the data model adds no persisted tables. The
review's least-privilege findings are addressed in R3 and R4, teardown in R10, request IDs and
timeouts in R8. Principle III remains pending until the full team approves the proposal.

## Project Structure

### Documentation (this feature)

```text
specs/001-platform-foundation/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── health.openapi.yaml
│   ├── error-response.schema.json
│   ├── log-entry.schema.json
│   └── required-checks.md
└── tasks.md             # Created by speckit-tasks on the Phase 1 branch, not here
```

### Source Code (repository root)

```text
package.json             # adds npm workspaces and root scripts (lint, typecheck, test, build, synth)
tsconfig.base.json
eslint.config.js
vitest.config.ts         # projects: apps/web, apps/api, infra; combined coverage ≥70% lines

apps/
├── web/                 # React + Vite placeholder page
│   ├── src/
│   └── tests/
└── api/                 # Lambda handlers
    ├── src/
    │   ├── health/      # GET /api/health
    │   └── lib/         # logger, withErrorHandling, db client, secret loader
    └── tests/

infra/                   # npm workspace with two separate CDK apps (TypeScript)
├── cdk.json             # application app: KanbanDev-* stacks, deployed by CI
├── bin/app.ts           # applies CiAccess-PermissionsBoundary and tag app=kanban-dev to all
├── lib/
│   ├── network-stack.ts     # VPC, isolated subnets, Secrets Manager endpoint
│   ├── data-stack.ts        # RDS PostgreSQL, generated secret, RDS CA trust
│   ├── auth-stack.ts        # Cognito user pool + SPA client
│   ├── api-stack.ts         # HTTP API, health Lambda (fromAsset), log groups
│   └── web-stack.ts         # S3 + CloudFront (/, /api/*), BucketDeployment
├── ci-access/           # separate CDK app, deployed manually by an admin, never by CI
│   ├── cdk.json
│   ├── bin/app.ts
│   └── lib/ci-access-stack.ts  # LegacyStackSynthesizer, no assets: OIDC provider (L1),
│                               # CiAccess-DeployRole, -CfnExecutionPolicy, -PermissionsBoundary
└── tests/

.github/workflows/
├── ci.yml               # required checks on pull requests (SHA-pinned actions)
├── deploy-dev.yml       # push to main + workflow_dispatch; concurrency group deploy-dev
└── destroy-dev.yml      # dispatch only, confirmation input; same concurrency group

docs/development.md      # setup, local checks, bootstrap, deploy/update/destroy (FR-012)
README.md                # GitHub workflows table and required-check list updated
```

**Structure Decision**: Web application layout under `apps/` plus a separate `infra/` CDK app,
managed as npm workspaces from the existing root `package.json`. The existing toolkit files
(`scripts/`, `.specify/`, `.agents/`) are unchanged.

## Complexity Tracking

No constitution violations require justification.
