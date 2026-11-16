# Research: Platform Foundation

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-11-16

Each decision below is part of the architecture proposal that FR-013 and constitution Principle III
require the full team to review before implementation.

## R1 — Overall architecture: serverless or containers

- **Decision**: Serverless. A React single-page app is hosted in a private S3 bucket behind
  CloudFront. The backend runs as AWS Lambda functions behind an API Gateway HTTP API, which
  CloudFront also serves under `/api/*`.
- **Rationale**: Principle V prefers serverless where practical. A dev environment with low,
  bursty traffic costs almost nothing at idle on Lambda. A single CloudFront domain serves both the
  page and the API, so no CORS setup is needed and HTTPS applies to both (FR-006a). There are no
  servers to patch, which keeps operations simple (Principle IX).
- **Alternatives considered**: ECS Fargate behind an ALB. It is rejected because of its constant
  idle cost and the extra load balancer and task-image pipeline. App Runner is rejected because it
  offers less control over the network path to RDS and is not a fit for event-driven work in later
  entries.

## R2 — Database and network path

- **Decision**: Amazon RDS for PostgreSQL 17 on `db.t4g.micro`, single-AZ, in private isolated
  subnets of a two-AZ VPC with no NAT gateway. The Lambda functions run in the same isolated
  subnets. They reach Secrets Manager through one interface VPC endpoint and connect to the
  database with `pg` (node-postgres) over TLS.
- **Rationale**: The constitution names PostgreSQL on Amazon RDS. With isolated subnets the
  database has no internet route at all. Removing the NAT gateway saves its constant cost
  (~$32/month); the Secrets Manager endpoint (~$7–15/month) is the only paid network item. A
  single-AZ micro instance is enough for a disposable dev environment (spec: data is disposable).
- **Alternatives considered**:
  - Aurora Serverless v2 with the Data API. It would need no VPC for the Lambda functions, but the
    Data API driver and its semantics differ from standard Postgres. The team would also have to
    agree that Aurora meets "PostgreSQL on Amazon RDS".
  - RDS Proxy. It is unnecessary at dev-environment concurrency, and the team can add it later
    without changing contracts.
  - A NAT gateway. It costs more and opens a broader egress path than this feature needs.

## R3 — Secrets and credentials

- **Decision**:
  - CDK generates the database master credentials into Secrets Manager. The health Lambda gets
    `secretsmanager:GetSecretValue` on that one secret ARN only and caches the value per execution
    environment.
  - CI deploys through GitHub Actions OIDC, which assumes a role scoped to this repository's `main`
    branch and the `dev` GitHub Environment. No long-lived AWS keys are stored.
  - GitHub secret scanning with push protection is enabled (the repository is public, so it is
    free), and a `gitleaks` step runs in CI.
- **Rationale**: This covers FR-008, FR-009 and the "secret added to a commit" edge case. The OIDC
  trust condition restricts who can deploy.
- **Alternatives considered**:
  - Long-lived IAM user keys in GitHub secrets. They violate FR-009.
  - SSM Parameter Store SecureString. It is workable, but Secrets Manager supports generation and
    rotation natively.

## R4 — Bootstrapping the deploy role

- **Decision**: A separate CDK stack, `DevCiAccessStack`, defines the GitHub OIDC provider and the
  deploy role. A team member with administrator access deploys it once, by hand, following
  `quickstart.md`. Every later change to it goes through a reviewed pull request and that same
  manual deploy. The deploy role may only assume the CDK bootstrap roles (`cdk-*-deploy-role`,
  `file-publishing`, `lookup`).
- **Rationale**: CI cannot create the role it needs in order to deploy. Keeping the role in code
  still satisfies Principle VI. Assuming only the CDK bootstrap roles is the least-privilege
  pattern CDK recommends.
- **Alternatives considered**: Creating the role by hand in the console. It is rejected under
  Principle VI.

## R5 — Monorepo and tooling

- **Decision**:
  - npm workspaces in the existing root `package.json`, with workspaces `apps/web`, `apps/api`
    and `infra`.
  - TypeScript 5.x in strict mode, with a shared `tsconfig.base.json`.
  - ESLint 9 flat config with `typescript-eslint`; Prettier, which is already present.
  - Vite for the web build.
  - esbuild through the CDK `NodejsFunction` for Lambda bundles.
- **Rationale**: This gives one install, one lockfile and one set of root scripts that CI and team
  members both run (US4), which is the least tooling that meets the constitution.
- **Alternatives considered**:
  - pnpm or Turborepo. They add a tool the team must learn, and npm is already used here.
  - Nx. It is too heavy for three packages.

## R6 — Testing and the coverage gate

- **Decision**:
  - Vitest with one root configuration that lists the three workspaces as projects. Coverage uses
    `@vitest/coverage-v8` with `thresholds.lines = 70`, computed over the combined source of
    `apps/*/src` and `infra/lib`.
  - Infrastructure is tested with CDK `Template` assertions.
  - The web app is tested with React Testing Library and jsdom.
- **Rationale**: This matches the clarification that coverage is lines at 70% or more over the
  whole repository combined (FR-003). A single run produces a single report, so the check is easy
  to read.
- **Alternatives considered**: Jest. It has slower ESM/TypeScript setup, and Vitest works directly
  with Vite.

## R7 — Runtime versions

- **Decision**: The Lambda runtime is the newest Node.js LTS that Lambda supports
  (`nodejs24.x`). Local development and CI use `.nvmrc`, and the CI matrix runs the checks on
  `.nvmrc` only.
- **Rationale**: The Lambda bundle is built by esbuild with `target: node24`. Using a newer Node
  locally does not change the deployed artifact.
- **Alternatives considered**: Pinning local tooling to Node 24. That changes the toolkit's
  existing `.nvmrc`, which is outside this feature's scope.

## R8 — Logging, request IDs and error responses

- **Decision**:
  - The backend logs with the AWS Lambda Powertools for TypeScript `Logger`, which writes JSON
    with timestamp, level, message, `requestId` (the API Gateway request ID) and `service`.
  - A small `withErrorHandling` wrapper catches unexpected errors, logs them at `error` level and
    returns the [error contract](./contracts/error-response.schema.json) with status 500.
  - Every response includes an `x-request-id` header.
  - Log groups are created explicitly with 7-day retention and `RemovalPolicy.DESTROY`.
- **Rationale**: This covers FR-010 and FR-011, US3 and SC-005: the request ID in the response is
  the search key in CloudWatch Logs Insights. Powertools is maintained by AWS and is
  dependency-light.
- **Alternatives considered**: `pino`. It works, but it needs manual Lambda context wiring.

## R9 — Deployment on merge and concurrency

- **Decision**:
  - The `deploy-dev.yml` workflow triggers on `push` to `main` and has
    `concurrency: { group: deploy-dev, cancel-in-progress: false }`.
  - It runs `cdk deploy --all --require-approval never` in the `dev` GitHub Environment, then runs
    a smoke check that calls `/api/health` through CloudFront and expects `healthy`.
  - The CloudFront invalidation covers `/index.html` only; assets have hashed names.
- **Rationale**: This meets FR-007 and the edge case about merges in quick succession: queued runs
  execute in order and the environment ends at the latest commit. CloudFormation rolls back a
  failed stack update automatically, which covers the edge case about a partial deploy.
- **Alternatives considered**: CDK Pipelines (CodePipeline). It adds a second CI system next to
  GitHub Actions.

## R10 — Teardown and data deletion

- **Decision**:
  - The RDS instance uses `removalPolicy: DESTROY` and `deleteAutomatedBackups: true`, with
    deletion protection off.
  - The S3 bucket uses `autoDeleteObjects: true`; log groups and the Cognito user pool use
    `DESTROY`.
  - `npm run destroy:dev` runs `cdk destroy` for the app stacks, but not for `DevCiAccessStack`.
- **Rationale**: This meets FR-012a and SC-006: no retained or billable leftovers, and a redeploy
  starts empty.
- **Alternatives considered**: Taking a final snapshot on destroy. That was rejected during
  clarification.

## R11 — Cognito in this entry

- **Decision**: Provision only the Cognito user pool, with email sign-in, self sign-up off, a
  strong password policy and `RemovalPolicy.DESTROY`, plus a public SPA app client without a
  secret. No sign-in UI and no API authorizer yet.
- **Rationale**: FR-005 requires the user directory to exist. Sign-in flows belong to entry 002
  (constitution Principle I: keep scope narrow).
- **Alternatives considered**: Deferring Cognito entirely to entry 002. It is rejected because
  FR-005 explicitly includes it.

## R12 — Required checks in the ruleset

- **Decision**: `ci.yml` has one job per required check: `Lint`, `Type-check`,
  `Unit tests and coverage`, `Build`, `CDK synth` and `Secret scan`. After the workflow has run once
  on a pull request, a repository admin adds these jobs to the `main` ruleset's required checks.
  The README's "Repository rulesets" list is updated in the same pull request.
- **Rationale**: This meets FR-001 and FR-002. Per the README, the ruleset change is a settings
  change recorded in the pull request.
- **Alternatives considered**: A single combined job. It is rejected because separate check names
  make failures visible at a glance (US1).
