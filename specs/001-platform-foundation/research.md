# Research: Platform Foundation

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-11-16

Each decision below is part of the architecture proposal that FR-013 and constitution Principle III
require the full team to review before implementation.

## R1 — Overall architecture: serverless or containers

- **Decision**: Serverless. A React single-page app is hosted in a private S3 bucket behind
  CloudFront. The backend runs as AWS Lambda functions behind an API Gateway HTTP API, which
  CloudFront also serves under `/api/*`.
- **CloudFront configuration**:
  - Default behavior → S3 origin with Origin Access Control and `redirect-to-https`.
  - `/api/*` behavior → HTTP API origin with `redirect-to-https` (CDK's default, `ALLOW_ALL`,
    would serve the API over plain HTTP), the `CachingDisabled` cache policy, the
    `AllViewerExceptHostHeader` origin request policy (forwarding `Host` makes API Gateway return
    403) and all HTTP methods allowed.
  - HTTP API routes include the `/api` prefix (for example `GET /api/health`).
  - A CDK assertion test checks that every cache behavior uses `redirect-to-https` (FR-006a).
  - There are no distribution-wide `customErrorResponses`, because they would hide API errors. SPA
    deep links are handled by a CloudFront Function on the default behavior only.
- **Rationale**: Principle V prefers serverless where practical. A dev environment with low,
  bursty traffic costs almost nothing at idle on Lambda. A single CloudFront domain serves both the
  page and the API, so no CORS setup is needed and HTTPS applies to both (FR-006a). There are no
  servers to patch, which keeps operations simple (Principle IX).
- **Alternatives considered**:
  - ECS Fargate behind an ALB. It is rejected because of its constant idle cost and the extra load
    balancer and task-image pipeline.
  - App Runner. It is rejected because it offers less control over the network path to RDS and is
    not a fit for event-driven work in later entries.
- **Known limitation**: the HTTP API's default `execute-api` endpoint also stays reachable (HTTPS
  only) without going through CloudFront. This is acceptable for a public health check; later
  entries that add authenticated routes enforce authorization in the API itself.

## R2 — Database and network path

- **Decision**:
  - Amazon RDS for PostgreSQL 17 on `db.t4g.micro`, single-AZ, with `storageEncrypted: true` set
    explicitly (CDK's `DatabaseInstance` default leaves storage unencrypted).
  - It sits in private isolated subnets of a two-AZ VPC with no NAT gateway. The Lambda functions
    run in the same isolated subnets and reach Secrets Manager through one interface VPC endpoint.
  - The functions connect with `pg` (node-postgres) over TLS. The Amazon RDS CA bundle ships with
    the Lambda bundle and `ssl.rejectUnauthorized` stays `true`; disabling certificate validation
    is not an acceptable fix for TLS errors. RDS PostgreSQL 17 requires SSL by default.
- **Rationale**: The constitution names PostgreSQL on Amazon RDS. With isolated subnets the
  database has no internet route at all. Removing the NAT gateway saves its constant cost
  (~$32/month); the Secrets Manager endpoint (~$7–15/month) is the only paid network item. A
  single-AZ micro instance is enough for a disposable dev environment (spec: data is disposable).
- **Alternatives considered**:
  - Aurora Serverless v2 with the Data API. It would need no VPC for the Lambda functions, but the
    Data API driver and its semantics differ from standard Postgres. The team would also have to
    agree that Aurora meets "PostgreSQL on Amazon RDS".
  - RDS Proxy. It is unnecessary at dev-environment concurrency and can be added later without
    changing contracts.
  - A NAT gateway. It costs more and opens a broader egress path than this feature needs.
- **Consequence for later entries**: Lambdas in the VPC can reach only AWS services that have a
  VPC endpoint. Each new service, such as the Cognito admin APIs, needs another interface endpoint,
  which is a cost decision for that entry.

## R3 — Secrets and deploy credentials

- **Decision**:
  - CDK generates the database master credentials into Secrets Manager. The health Lambda gets
    `secretsmanager:GetSecretValue` on that one secret ARN only and caches the value per execution
    environment.
  - CI deploys through GitHub Actions OIDC; no long-lived AWS keys are stored. Three controls
    limit who can deploy:
    1. The deploy role's trust policy requires `aud = sts.amazonaws.com` and
       `sub = repo:OsKuik/spec-toolkit:environment:dev`.
    2. The `dev` GitHub Environment's deployment branch policy allows **only `main`**. Because the
       `sub` claim of an environment job carries no branch, this policy, not the trust policy, is
       what stops a workflow on another branch from assuming the role.
    3. The `main` ruleset requires pull requests and the required checks, so workflow changes reach
       `main` only after review.
  - GitHub secret scanning with push protection is enabled (the repository is public, so it is
    free), and a `gitleaks` job runs in CI. The existing `lefthook` pre-commit secret hook also
    applies locally.
- **Rationale**: This covers FR-008, FR-009 and the "secret added to a commit" edge case.
- **Alternatives considered**:
  - Long-lived IAM user keys in GitHub secrets. They violate FR-009.
  - SSM Parameter Store SecureString. It is workable, but Secrets Manager supports generation and
    rotation natively.
  - Customizing the OIDC subject template to include `ref`. It is not needed while the environment
    branch policy is in place; it remains an option if the team wants defense in depth.

## R4 — Deploy role, CDK bootstrap and least privilege

- **Decision**: The deploy access lives in a **separate CDK app**, `infra/ci-access/`, with its own
  entry point and `cdk.json`. It is never part of the application app, so CI can never deploy or
  change its own permissions. A team member with administrator access deploys it by hand, and every
  change to it goes through a reviewed pull request first.
  - **No bootstrap needed**: the ci-access app uses `LegacyStackSynthesizer`, contains no assets
    and no Lambda-backed custom resources, and keeps its template under 50 KB. So it deploys before
    `cdk bootstrap` exists, which resolves the ordering problem: bootstrap needs the execution
    policy ARN that this app creates. A CDK assertion test checks that the synthesized template
    has no assets and no `Custom::` resources.
  - **Naming**: every ci-access resource uses the `CiAccess-` prefix, kept separate from the
    `KanbanDev-*` names that CI manages.
- **The ci-access app defines**:
  - **OIDC provider**: the GitHub OIDC provider, created with the native L1 `iam.CfnOIDCProvider`
    and not the custom-resource-backed L2. If the account already has a provider for
    `token.actions.githubusercontent.com`, set the context flag `useExistingOidcProvider=true`. The
    app then references the predictable ARN
    `arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com` instead of creating
    one, and synth needs no credentials either way.
  - **`CiAccess-DeployRole`**, the deploy role (R3 trust). It may:
    - assume the CDK bootstrap roles `cdk-*-deploy-role` and `cdk-*-file-publishing-role` (no
      lookup role, because the stacks do no context lookups, R12);
    - for the R10 teardown, call `logs:DescribeLogGroups`, call `logs:DeleteLogGroup` on
      `/aws/lambda/KanbanDev-*`, and run the read-only checks: `tag:GetResources`,
      `rds:DescribeDBInstances`, `rds:DescribeDBSnapshots`,
      `rds:DescribeDBInstanceAutomatedBackups`, `cloudfront:ListDistributions`,
      `s3:ListAllMyBuckets` and `secretsmanager:DescribeSecret`.
  - **`CiAccess-CfnExecutionPolicy`**, a customer-managed policy that replaces CDK's default
    `AdministratorAccess` for the CloudFormation execution role. It allows only the services the
    app stacks use: CloudFormation, EC2 (VPC, subnets, security groups, endpoints), RDS, Secrets
    Manager, Lambda, API Gateway, CloudFront, S3, Cognito, CloudWatch Logs and SSM bootstrap
    parameters. S3 write actions are limited to buckets named `kanbandev-*`, so the policy cannot
    change the `CDKToolkit` staging bucket or other buckets. Its IAM rules:
    - IAM write actions are limited to roles named `KanbanDev-*` (CDK names roles after their
      stack, for example `KanbanDev-Api-HealthFnServiceRole…`). `iam:PassRole` on those roles is
      allowed only to `lambda.amazonaws.com`.
    - `iam:CreateRole`, `iam:PutRolePolicy`, `iam:AttachRolePolicy` and
      `iam:PutRolePermissionsBoundary` require the condition
      `iam:PermissionsBoundary = arn:aws:iam::<account>:policy/CiAccess-PermissionsBoundary`
      (the condition takes the full ARN).
    - Explicit `Deny`, which no allow can override:
      - `iam:DeleteRolePermissionsBoundary` on any role;
      - every IAM write action, including `iam:CreatePolicyVersion`,
        `iam:SetDefaultPolicyVersion`, `iam:DeletePolicy` and `iam:UpdateAssumeRolePolicy`, on
        `CiAccess-*` roles and policies and on the OIDC provider;
      - `iam:CreateUser`, `iam:CreateAccessKey` and `iam:CreatePolicy`.
    - `iam:CreateServiceLinkedRole` is allowed only with
      `iam:AWSServiceName = rds.amazonaws.com`, which the first RDS deploy in a fresh account needs.
  - **`CiAccess-PermissionsBoundary`**, a managed policy that the application app applies to every
    role it creates through `PermissionsBoundary.of(app)`. It caps those roles at what the
    application and CDK's helper functions need:
    - CloudWatch Logs: `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`.
    - Lambda VPC networking: `ec2:CreateNetworkInterface`, `ec2:DescribeNetworkInterfaces`,
      `ec2:DeleteNetworkInterface`, `ec2:AssignPrivateIpAddresses`,
      `ec2:UnassignPrivateIpAddresses`, `ec2:DescribeSubnets` (the actions in AWS's
      `AWSLambdaVPCAccessExecutionRole`).
    - The VPC default-security-group restriction helper: `ec2:AuthorizeSecurityGroupIngress`,
      `ec2:AuthorizeSecurityGroupEgress`, `ec2:RevokeSecurityGroupIngress`,
      `ec2:RevokeSecurityGroupEgress`.
    - `secretsmanager:GetSecretValue`.
    - S3 for `BucketDeployment` and auto-delete: `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`,
      `s3:ListBucket`, `s3:GetBucketTagging`, `s3:PutBucketPolicy`, plus `s3:ListBucketVersions`
      and `s3:DeleteObjectVersion`. CDK's auto-delete handler lists and deletes object versions
      even on an unversioned bucket; without these, destroy cannot empty the bucket.
    - `cloudfront:CreateInvalidation`, `cloudfront:GetInvalidation`.
    - `kms:Decrypt` with the condition `kms:ViaService` in `s3.<region>.amazonaws.com` or
      `secretsmanager.<region>.amazonaws.com`. The CDK staging bucket and the database secret
      are encrypted by default with the AWS-managed `aws/s3` and `aws/secretsmanager` keys, and a
      boundary may also limit permissions that a key policy grants. `BucketDeployment` needs this
      to read the web assets, and the health function needs it to read the secret. The first dev
      deploy and the smoke check confirm this.
    - No IAM actions at all.

    Each role's own policy still narrows these to specific resources; the boundary is the upper
    limit if a role policy is ever too broad. CDK assertion tests catch a missing action in CI
    rather than at deploy or destroy time:
    - **Helpers**: CDK's helper grants are wildcards (`s3:GetObject*`, `s3:List*`, …), some
      live in the bucket policy rather than a role policy, and managed policies appear only as
      ARNs. So the test keeps a pinned list of the API actions each helper calls and asserts that
      the boundary contains all of them:
      - auto-delete: `s3:ListBucketVersions`, `s3:DeleteObjectVersion`, `s3:GetBucketTagging`,
        `s3:PutBucketPolicy`;
      - `BucketDeployment`: S3 get, put, list and delete, `kms:Decrypt`,
        `cloudfront:CreateInvalidation` and `cloudfront:GetInvalidation`;
      - default-security-group restriction: the four security-group actions;
      - VPC Lambdas: the network-interface actions;
      - health function: `secretsmanager:GetSecretValue` and `kms:Decrypt`.

      The pinned list is re-checked on every `aws-cdk-lib` upgrade. A helper action CDK adds
      later is otherwise caught only at runtime, by the deploy smoke check or a destroy run.
    - **Application roles**: selected by construct path (for example the health function's
      role), not by their content. The test asserts that their policies contain no wildcard
      actions, and that every action is in the boundary.
- **Bootstrap**: after the ci-access app is deployed, the account is bootstrapped once with
  `cdk bootstrap --cloudformation-execution-policies <CiAccess-CfnExecutionPolicy ARN>`.
- **Rationale**: This addresses Principle IV's least privilege beyond the deploy role. Even a
  compromised `main` workflow can create only `KanbanDev-*` roles, which are capped by a boundary
  that grants no IAM actions. It cannot remove or replace that boundary, and it cannot touch the
  `CiAccess-*` resources. Keeping the access app separate removes the self-modification path.
- **Alternatives considered**:
  - Default `cdk bootstrap`, which gives the execution role `AdministratorAccess`. It was rejected
    by the team during review.
  - Defining the role in the application app behind a context flag. It is easy to deploy by
    accident with `--all`.
  - Creating the role by hand in the console. It is rejected under Principle VI.
- **Security review**: the ci-access app, the execution policy and the boundary are
  security-sensitive (Principle IV). They need a documented `security-auditor` review before their
  first deploy.

## R5 — Monorepo and tooling

- **Decision**:
  - npm workspaces in the existing root `package.json`, with workspaces `apps/web`, `apps/api` and
    `infra`.
  - TypeScript 5.x in strict mode, with a shared `tsconfig.base.json`.
  - ESLint 9 flat config with `typescript-eslint`; Prettier, which is already present.
  - Vite for the web build and esbuild for the Lambda bundles.
  - Versions are pinned to the latest stable releases when the project is scaffolded.
- **Rationale**: This gives one install, one lockfile and one set of root scripts that CI and team
  members both run (US4), which is the least tooling that meets the constitution.
- **Alternatives considered**:
  - pnpm or Turborepo. They add a tool the team must learn, and npm is already used here.
  - Nx. It is too heavy for three packages.

## R6 — Testing and the coverage gate

- **Decision**:
  - Vitest with one root configuration that lists the three npm workspaces as projects. Coverage
    uses `@vitest/coverage-v8` with `thresholds.lines = 70`, computed over all application and
    infrastructure TypeScript source combined: `apps/web/src`, `apps/api/src`, `infra/bin`, `infra/lib`
    and `infra/ci-access`. Toolkit scripts are excluded (spec clarification, FR-003).
  - Infrastructure is tested with CDK `Template` assertions. These include:
    - every application Lambda and the `BucketDeployment` function has an explicit log group with
      7-day retention. The `BucketDeployment` function is checked by name, because it is a
      singleton function that also carries custom-resource-handler metadata.
      - Only CDK's `CustomResourceProvider` helpers are excluded: auto-delete objects and the
        default-security-group restriction. They create their own log groups at runtime, and R10
        sweeps them.
      - They are found through the construct metadata
        `aws:cdk:is-custom-resource-handler-customResourceProvider`, which is in the cloud
        assembly (`stackArtifact.findMetadataByType`), not in the CloudFormation template. The
        test maps those construct paths to logical IDs;
    - every IAM role carries the permissions boundary;
    - every CloudFront behavior redirects HTTP to HTTPS.
  - The web app is tested with React Testing Library and jsdom.
- **Rationale**: This matches the clarified coverage scope. A single run produces a single report,
  so the check is easy to read.
- **Alternatives considered**: Jest. It has slower ESM/TypeScript setup, and Vitest works directly
  with Vite.

## R7 — Runtime versions

- **Decision**:
  - The Lambda runtime is `nodejs24.x`, the newest Node.js runtime generally available on Lambda.
    `nodejs26.x` is in public preview as of November 2026; switch when it becomes generally
    available.
  - Local development and CI use `.nvmrc` (Node 26). Because esbuild's `target: node24` lowers
    syntax but not runtime APIs, CI also runs the `apps/api` tests on Node 24.
- **Rationale**: The API test job on Node 24 catches runtime APIs that exist in Node 26 but not in
  Node 24, so the toolkit's `.nvmrc` stays unchanged.
- **Alternatives considered**: Pinning local tooling to Node 24. That changes the toolkit's
  existing `.nvmrc`, which is outside this feature's scope.

## R8 — Logging, request IDs, error responses and timeouts

- **Decision**:
  - **Logger**: the backend logs with the AWS Lambda Powertools for TypeScript `Logger`, which
    writes JSON.
  - **Request ID**: Powertools logs the Lambda `function_request_id` by default but not the API
    Gateway request ID. So the `withErrorHandling` wrapper calls
    `logger.appendKeys({ requestId: event.requestContext.requestId })` at the start of every
    invocation. It uses the same value for the `x-request-id` response header and for
    `error.requestId`, which makes that one ID the search key in Logs Insights (SC-005).
  - **Errors**: the wrapper catches unexpected errors, logs them at `ERROR` level and returns the
    [error contract](./contracts/error-response.schema.json) with status 500.
  - **Timeouts**: the health check's database call is bounded well below the Lambda timeout:
    - Secrets Manager client: 2 s request timeout, with `maxAttempts: 1` (the SDK default of 3
      attempts would exceed the budget).
    - `pg`: `connectionTimeoutMillis: 2000` and `query_timeout: 2000`.
    - Lambda: 10 s.
    - Worst case: 2 s secret fetch (first call only; cached afterward) plus 2 s connect plus 2 s
      query, about 6 s, which stays inside the Lambda timeout even with a VPC cold start.

    When the database is unreachable, the function therefore returns `503 {"status":"unhealthy"}`
    with `x-request-id` instead of timing out into an API Gateway 500.
  - **Log groups**: created explicitly, with 7-day retention and `RemovalPolicy.DESTROY`
    (`LogGroup` plus the function's `logGroup` property). CDK's deprecated `logRetention` helper is
    not used.
- **Rationale**: This covers FR-006, FR-010, FR-011, US3 and SC-005. Powertools is maintained by
  AWS and is dependency-light.
- **Alternatives considered**:
  - `pino`. It works, but it needs manual Lambda context wiring.
  - Using Powertools' `correlation_id`. It is workable, but `requestId` keeps the contract
    self-explanatory.

## R9 — Deployment on merge and concurrency

- **Decision**:
  - `deploy-dev.yml` triggers on `push` to `main` and on `workflow_dispatch`. The `dev` Environment
    restricts both to `main` (R3).
  - It has `concurrency: { group: deploy-dev, cancel-in-progress: false }`.
  - It runs `npm ci`, `npm run build`, then `cdk deploy KanbanDev-Network KanbanDev-Data
    KanbanDev-Auth KanbanDev-Api KanbanDev-Web --require-approval never`. Stacks are named
    explicitly; the application app holds only these stacks.
  - It then runs a smoke check that calls `/api/health` through CloudFront and expects `healthy`.
  - Web assets reach S3 through CDK `BucketDeployment` (with an explicit log group). It invalidates
    `/index.html` through its `distribution` and `distributionPaths` properties, so the deploy role
    needs no CloudFront permissions; other assets have hashed names.
  - Manual deploys, including the first one after bootstrap, use **workflow dispatch** of the same
    workflow, never a local `cdk deploy`. So every deploy shares the concurrency group and deploys
    exactly one commit on `main`.
- **Behavior**: GitHub keeps at most one pending run per concurrency group, and a newer pending run
  replaces an older one. Intermediate commits may therefore be skipped, but the environment always
  ends at the latest merged commit (FR-007). CloudFormation rolls back a failed stack update. With
  several stacks, stacks that had already finished stay on the new commit; the environment stays
  redeployable, and the next successful run converges it.
- **Alternatives considered**: CDK Pipelines (CodePipeline). It adds a second CI system next to
  GitHub Actions.

## R10 — Teardown and data deletion

- **Decision**:
  - **Resources**:
    - The RDS instance uses `removalPolicy: DESTROY` and `deleteAutomatedBackups: true`, with
      deletion protection off.
    - The S3 bucket uses `removalPolicy: DESTROY` together with `autoDeleteObjects: true`.
    - Log groups and the Cognito user pool use `DESTROY`.
  - **Workflow**: a `destroy-dev.yml` workflow (dispatch only; requires typing `destroy-dev` as a
    confirmation input; same `deploy-dev` concurrency group and `dev` Environment) runs
    `cdk destroy` on the five app stacks.
  - **Sweep**: it then deletes any remaining `/aws/lambda/KanbanDev-*` log groups. CDK-internal
    helpers (the auto-delete-objects provider and the VPC default-security-group restriction
    provider) create these at runtime outside CloudFormation.
  - **Verification**: the workflow finishes with a check that retries for up to 10 minutes,
    because deleted resources can stay visible briefly, and fails if any of these remain:
    - resources tagged `app=kanban-dev` (`tag:GetResources` in the app region, and in `us-east-1`
      for CloudFront). Secrets Manager secrets that are scheduled for deletion are excluded: a
      deleted secret can stay visible with its tags during its recovery window, and it is not
      billed. `secretsmanager:DescribeSecret` shows its `DeletedDate`. Implementation verifies
      this behavior in the dev account. The RDS secret keeps a CDK-generated name, so a redeploy
      never collides with a secret that is still pending deletion;
    - RDS instances, snapshots or automated backups whose identifier starts with `kanbandev`;
    - CloudFront distributions whose comment is `KanbanDev` (set on the distribution in the web
      stack);
    - S3 buckets with the `kanbandev-` prefix;
    - `/aws/lambda/KanbanDev-*` log groups.
- **Exclusions**: the ci-access app and the CDK bootstrap stack (`CDKToolkit` and its staging
  bucket) are shared account setup and are intentionally kept. FR-012a's "no retained resources"
  applies to the application stacks.
- **Rationale**: This meets FR-012a and SC-006. Every CloudFormation-managed resource is deleted,
  the sweep handles helper log groups that CloudFormation never owned, and the redeploy starts
  empty. Lambda ENIs in a VPC can take a while to release, so destroy may take 20+ minutes.
- **Alternatives considered**:
  - A final snapshot on destroy. It was rejected during clarification.
  - Disabling the helper custom resources. Dropping the default-security-group restriction weakens
    security, and dropping auto-delete makes destroy fail on a non-empty bucket.

## R11 — Cognito in this entry

- **Decision**: Provision only the Cognito user pool, with email sign-in, self sign-up off, a
  strong password policy and `RemovalPolicy.DESTROY`, plus a public SPA app client without a
  secret. No sign-in UI and no API authorizer yet.
- **Rationale**: The user pool is a shared, security-sensitive environment resource. Creating it
  with the foundation means it goes through the same review as the other baseline resources:
  permissions boundary, teardown behavior and the security review of the ci-access setup. It also
  gives entry 002 a stable pool to build sign-in flows on, without changing the network and
  deploy structure. Sign-in flows stay in entry 002 (Principle I).
- **Alternatives considered**: Deferring the pool entirely to entry 002. That is simpler for this
  entry, but it moves a new security-sensitive resource type into a feature PR. It also conflicts
  with FR-005 in the approved spec.

## R12 — Required checks and repository settings

- **Decision**: `ci.yml` runs on every pull request with one job per required check:
  - `Lint`
  - `Type-check`
  - `Unit tests and coverage`
  - `API tests (Node 24)`
  - `Build`
  - `CDK synth`
  - `Secret scan`

  Required jobs have **no job-level `if:` conditions and no path filters**. GitHub treats a skipped
  required job as passing, so skipping would bypass FR-002. A missing job blocks merge as
  "waiting for status".
- **Workflow rules**:
  - All `uses:` references are pinned to full commit SHAs, as `scripts/test_workflow_action_pins.py`
    enforces.
  - The `CDK synth` job runs without AWS credentials. Stacks are environment-agnostic (no `env`, no
    context lookups; AZs come from `Fn.getAzs`).
  - Lambda code comes from the `apps/api` esbuild output through `lambda.Code.fromAsset`. So
    `npm run synth` runs after `npm run build`, needs no Docker and bundles nothing twice.
- **Ruleset**: after `ci.yml` has run once on a pull request, a repository admin adds these jobs to
  the `main` ruleset's required checks. The same implementation pull request updates the README's
  "GitHub workflows" table and the "Repository rulesets" required-check list.
- **Rationale**: This covers FR-001 and FR-002. Per the README, the ruleset change is a settings
  change recorded in the pull request.
- **Alternatives considered**:
  - A single combined job. It is rejected because separate check names make failures visible at a
    glance (US1).
  - `NodejsFunction` bundling at synth time. It bundles twice and falls back to Docker when esbuild
    is missing.
