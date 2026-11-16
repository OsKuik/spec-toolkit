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
  type-check, unit tests with a 70% combined line-coverage gate, build, CDK synth) plus a secret
  scan;
- a serverless AWS dev environment: CloudFront in front of S3 for the web page, an API Gateway
  HTTP API with Lambda for `/api/*`, RDS for PostgreSQL in isolated subnets, and a Cognito user
  pool;
- automatic deploys on every merge to `main` through GitHub OIDC;
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
- Infrastructure: AWS CDK v2 (`aws-cdk-lib`).
- Tooling: ESLint 9 with `typescript-eslint`, Prettier, Vitest, and `gitleaks` in CI.

**Storage**: Amazon RDS for PostgreSQL 17 (`db.t4g.micro`, single-AZ, isolated subnets). No
application tables in this entry (R2)

**Testing**: Vitest with workspace projects and `@vitest/coverage-v8` (lines ≥ 70% combined),
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
- Least-privilege IAM.
- 7-day log retention.
- Destroy leaves nothing retained (FR-012a).
- Deploys to the dev environment never run concurrently (FR-007).

**Scale/Scope**: One shared dev environment, a team of a few developers, and three workspaces;
a health endpoint and a placeholder page only

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

| Principle | Gate | Status |
| --- | --- | --- |
| I. Narrow, Explicit Scope | No Kanban features; nothing from the out-of-scope list | Pass: placeholder page, health check, user pool only |
| II. Spec-Driven Development | Approved spec; every module traces to an FR | Pass: spec clarified and approved; mapping below |
| III. Team-Based Decisions | Architecture proposal reviewed by the full team before implementation | **Pending**: [research.md](./research.md) is the proposal; the spec-and-plan PR is its review venue. Implementation is blocked until every member has explicitly approved it |
| IV. Security-First | Documented security review before deploy; server-side authZ; no secrets in repo; least privilege | Pass by design: OIDC, scoped secret access, isolated DB, secret scanning. The deploy role and IAM changes are security-sensitive: run `security-auditor` on the implementation PR before the first deploy |
| V. Technology Baseline | TypeScript, React, RDS PostgreSQL, Cognito, AWS-managed, serverless preferred | Pass |
| VI. Real Deployment Target | Validated in AWS dev; CDK TypeScript; reproducible | Pass: deploy on merge, smoke check, destroy/redeploy in quickstart |
| VII. Quality Gate | Lint, type-check, unit tests, build, CDK synth; ≥70% coverage | Pass: one CI job per check (R12) |
| VIII. Git Workflow | PRs only; required checks before merge | Pass: required checks are added to the `main` ruleset. Note: `required_approvals = 0` in `project-context.toml` conflicts with this principle's 1 approval and is outside this feature |
| IX. Simplicity and Operational Clarity | Simplest design; small increments; logging/error handling | Pass: no NAT, no RDS Proxy, no CDK Pipelines; Powertools logging |
| X. Traceable Work Tracking | Work tied to issues | Pass: issue #4 for this PR; Phase issues come from `tasks.md` |

Requirement traceability:

| Requirement | Where it is implemented |
| --- | --- |
| FR-001, FR-002, FR-003 | `.github/workflows/ci.yml`, `vitest.config.ts`, ruleset |
| FR-004, FR-012a | `infra/` stacks |
| FR-005, FR-006, FR-006a | `infra/lib/*`, `apps/api/src/health`, `apps/web` |
| FR-007 | `.github/workflows/deploy-dev.yml` |
| FR-008, FR-009 | `infra/lib/data-stack.ts`, `infra/lib/ci-access-stack.ts`, `ci.yml` secret scan |
| FR-010, FR-011 | `apps/api/src/lib/`, `infra/lib/api-stack.ts` |
| FR-012 | `README.md` / `docs/development.md`, [quickstart.md](./quickstart.md) |
| FR-013 | This plan and [research.md](./research.md), reviewed in the spec-and-plan PR |

**Post-design re-check (after Phase 1)**: no new violations. The contracts add no endpoints beyond
`/api/health`, and the data model adds no persisted tables. Principle III remains pending until
the full team approves the proposal.

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

infra/                   # AWS CDK app (TypeScript)
├── bin/app.ts
├── lib/
│   ├── network-stack.ts     # VPC, isolated subnets, Secrets Manager endpoint
│   ├── data-stack.ts        # RDS PostgreSQL, generated secret
│   ├── auth-stack.ts        # Cognito user pool + SPA client
│   ├── api-stack.ts         # HTTP API, health Lambda, log groups
│   ├── web-stack.ts         # S3 + CloudFront (/, /api/*)
│   └── ci-access-stack.ts   # GitHub OIDC provider + deploy role (deployed manually once)
└── tests/

.github/workflows/
├── ci.yml               # required checks on pull requests
└── deploy-dev.yml       # deploy on push to main, concurrency group deploy-dev

docs/development.md      # setup, local checks, deploy/update/destroy (FR-012)
```

**Structure Decision**: Web application layout under `apps/` plus a separate `infra/` CDK app,
managed as npm workspaces from the existing root `package.json`. The existing toolkit files
(`scripts/`, `.specify/`, `.agents/`) are unchanged.

## Complexity Tracking

No constitution violations require justification.
