# Quickstart: Platform Foundation

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Validation guide proving the foundation works end to end. Each scenario names the requirement or
success criterion it validates.

## Prerequisites

- Node.js from `.nvmrc`, npm, Git, the GitHub CLI (`gh`).
- AWS CLI v2. Administrator access to the team's dev account is needed only for the one-time
  bootstrap in step 2 and the manual database stop in step 6; every deploy and destroy runs
  through GitHub Actions.
- Optional: `gitleaks` for local secret scanning.

## 1. Local checks (US1, US4, SC-007)

```bash
npm ci
npm run lint && npm run typecheck && npm run test:coverage && npm run build && npm run synth
```

Expected: every command exits 0, the coverage summary shows lines ≥ 70%, and `synth` works without
AWS credentials.

## 2. One-time account bootstrap (FR-009, R3, R4)

Done once per account and region by a team member with administrator access, after the
documented `security-auditor` review of the ci-access app (Principle IV):

1. Deploy the separate ci-access app. It needs no CDK bootstrap. It creates the OIDC provider
   (add `-c useExistingOidcProvider=true` if the account already has one), `CiAccess-DeployRole`,
   `CiAccess-CfnExecutionPolicy` and `CiAccess-PermissionsBoundary`:

   ```bash
   npm run deploy:ci-access
   ```

2. Bootstrap CDK with the scoped execution policy instead of `AdministratorAccess`:

   ```bash
   npx cdk bootstrap aws://<account>/<region> \
     --cloudformation-execution-policies <CiAccess-CfnExecutionPolicy ARN from step 1 outputs>
   ```

3. In GitHub, create the `dev` Environment:
   - Set its **deployment branches** to `main` only.
   - Set its variables `AWS_ACCOUNT_ID`, `AWS_REGION` and `AWS_DEPLOY_ROLE_ARN` (from the step 1
     outputs). They are not secrets.

4. Enable secret scanning push protection on the repository.

## 3. Required checks block merge (FR-001–FR-003, SC-001)

1. Open a pull request with a clean change. Every job in
   [required-checks.md](./contracts/required-checks.md) passes.
2. Push, one at a time:
   - a lint error;
   - a type error;
   - a failing test;
   - a change that drops coverage below 70%;
   - a use of a Node 26-only API in `apps/api`;
   - a broken CDK construct.

   Each time, the matching job fails and the merge button is blocked.
3. Push a commit containing a fake AWS key. `Secret scan` fails, or push protection rejects the
   push.
   - The local pre-commit secret hook blocks the commit first, so this test needs
     `git commit --no-verify`.
   - Use a throwaway branch: the commit stays in the public pull request history.

## 4. First deploy and deploy on merge (FR-004, FR-005, FR-007, SC-002, SC-003)

1. Run the first deploy through workflow dispatch from `main`:

   ```bash
   gh workflow run deploy-dev.yml --ref main
   ```

   The run deploys the five `KanbanDev-*` stacks, and its smoke step passes.
2. Check the health endpoint:

   ```bash
   curl -si https://<cloudfront-domain>/api/health
   ```

   Expected:
   - `HTTP/2 200`
   - an `x-request-id` header
   - body `{"status":"healthy"}`, with no other fields ([contract](./contracts/health.openapi.yaml))

   Opening `https://<cloudfront-domain>/` shows the placeholder page.
3. Merge a pull request to `main`. `deploy-dev.yml` runs automatically and the environment reflects
   that commit.
4. Merge two pull requests in quick succession. The deploys never overlap, and the environment ends
   at the later commit.
5. Run `gh workflow run deploy-dev.yml --ref <feature-branch>`. The `dev` Environment rejects it,
   because only `main` may deploy.

## 5. HTTPS only (FR-006a)

```bash
curl -si http://<cloudfront-domain>/api/health
```

Expected: a `301` redirect to `https://`.

## 6. Unhealthy and error paths (FR-006, FR-011, US3, SC-005)

1. Make the database unreachable. Use an administrator session, because the deploy role cannot do
   this, and run `aws rds stop-db-instance ...`; stopping takes several minutes.
2. Call `/api/health` again. Expected: within 10 s, `503` with body `{"status":"unhealthy"}` and
   an `x-request-id` header. It must not be an API Gateway 500 (R8 timeouts).
3. Copy the `x-request-id` value and query CloudWatch Logs Insights on the health function's log
   group:
   `fields @timestamp, level, message, requestId | filter requestId = "<id>"`.
   Expected: an entry matching [log-entry.schema.json](./contracts/log-entry.schema.json) with the
   cause, found in under 5 minutes.
4. Start the instance again.

The unexpected-error path (500 with
[error-response.schema.json](./contracts/error-response.schema.json)) is covered by unit tests of
`withErrorHandling`.

## 7. Security spot checks (FR-008, FR-009, FR-010)

- Run `git grep -nE 'AKIA[0-9A-Z]{16}'`. It finds only documented test fixtures, if any, and no
  real credentials.
- The health Lambda's role allows `secretsmanager:GetSecretValue` on one secret ARN only, and it
  carries `CiAccess-PermissionsBoundary`.
- The deploy role trust policy requires `repo:OsKuik/spec-toolkit:environment:dev`. The `dev`
  Environment allows only `main`.
- The CDK CloudFormation execution role has `CiAccess-CfnExecutionPolicy`, not
  `AdministratorAccess`.
- The log groups show a retention of 7 days, and recent entries contain no secret values.

## 8. Destroy and redeploy (FR-012a, SC-006)

1. Destroy the environment:

   ```bash
   gh workflow run destroy-dev.yml --ref main -f confirm=destroy-dev
   ```

   Expected:
   - The five app stacks are deleted.
   - Leftover `/aws/lambda/KanbanDev-*` log groups are swept.
   - The final check (R10) finds no remaining application resources: no RDS instance, snapshot,
     automated backup, CloudFront distribution, S3 bucket or log group.
   - The ci-access app and `CDKToolkit` remain by design.
   - Destroy can take 20+ minutes while Lambda network interfaces are released.
2. Rerun step 4.1. Step 4.2 passes again, against an empty database.

## 9. Onboarding (US4, SC-002)

A team member who did not build the foundation follows `docs/development.md` from a fresh clone.
They complete step 1, and then the step 4.1 dispatch, in under 30 minutes. This assumes step 2 has
already been done for the account.
