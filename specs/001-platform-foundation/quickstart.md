# Quickstart: Platform Foundation

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Validation guide proving the foundation works end to end. Each scenario names the requirement or
success criterion it validates.

## Prerequisites

- Node.js from `.nvmrc`, npm, Git, the GitHub CLI (`gh`).
- AWS CLI v2 with access to the team's dev account; administrator access only for the one-time
  bootstrap (step 2).
- Optional: `gitleaks` for local secret scanning.

## 1. Local checks (US1, US4, SC-007)

```bash
npm ci
npm run lint && npm run typecheck && npm run test:coverage && npm run build && npm run synth
```

Expected: every command exits 0, and the coverage summary shows lines ≥ 70%.

## 2. One-time account bootstrap (FR-009, R4)

Done once per account and region by a team member with administrator access:

```bash
npx cdk bootstrap aws://<account>/<region>
npm run deploy:ci-access   # deploys DevCiAccessStack (GitHub OIDC provider + deploy role)
```

Then, in GitHub:

1. Create a `dev` Environment.
2. Set its variables `AWS_ACCOUNT_ID`, `AWS_REGION` and `AWS_DEPLOY_ROLE_ARN` (from the stack
   output). They are not secrets.
3. Enable secret scanning push protection.

## 3. Required checks block merge (FR-001–FR-003, SC-001)

1. Open a pull request with a clean change: all jobs in
   [required-checks.md](./contracts/required-checks.md) pass.
2. Push, one at a time, a lint error, a type error, a failing test, a change that drops coverage
   below 70%, and a broken CDK construct. The matching job fails and the merge button is blocked.
3. Push a commit containing a fake AWS key: `Secret scan` fails (or push protection rejects it).

## 4. Deploy on merge (FR-004, FR-005, FR-007, SC-003)

Merge a pull request to `main`. The `deploy-dev.yml` run deploys and its smoke step passes.
Then:

```bash
curl -si https://<cloudfront-domain>/api/health
```

Expected:

- `HTTP/2 200`
- an `x-request-id` header
- body `{"status":"healthy"}`, with no other fields ([contract](./contracts/health.openapi.yaml))

Opening `https://<cloudfront-domain>/` shows the placeholder page. Merge two pull requests in quick
succession: the second deploy waits for the first, and the environment ends at the later commit.

## 5. HTTPS only (FR-006a)

```bash
curl -si http://<cloudfront-domain>/api/health
```

Expected: a `301` redirect to `https://`.

## 6. Unhealthy and error paths (FR-006, FR-011, US3, SC-005)

1. Temporarily stop the RDS instance (`aws rds stop-db-instance ...`) and call `/api/health`.
   Expected: `503` with body `{"status":"unhealthy"}`.
2. Copy the `x-request-id` value and query CloudWatch Logs Insights on the health function's log
   group:
   `fields @timestamp, level, message, requestId | filter requestId = "<id>"`. Expected: an entry
   matching [log-entry.schema.json](./contracts/log-entry.schema.json) with the cause, found in
   under 5 minutes. Start the instance again.
3. The unexpected-error path (500 with [error-response.schema.json](./contracts/error-response.schema.json))
   is covered by unit tests of `withErrorHandling`.

## 7. Security spot checks (FR-008, FR-009, FR-010)

- `git grep -nE 'AKIA|password\s*='` finds no credentials.
- The health Lambda's role allows `secretsmanager:GetSecretValue` on one secret ARN only.
- The deploy role trust policy requires `repo:OsKuik/spec-toolkit:environment:dev`.
- The log groups show a retention of 7 days, and recent entries contain no secret values.

## 8. Destroy and redeploy (FR-012a, SC-006)

```bash
npm run destroy:dev
```

Expected:

- The app stacks are deleted; `DevCiAccessStack` remains.
- No RDS instance, snapshot or automated backup remains.
- No retained S3 bucket or log group remains.

Re-run `deploy-dev.yml` (workflow dispatch) or `npm run deploy:dev`. Step 4 passes again against
an empty database.

## 9. Onboarding (US4, SC-002)

A team member who did not build the foundation follows `docs/development.md` from a fresh clone
through steps 1 and 4 in under 30 minutes.
