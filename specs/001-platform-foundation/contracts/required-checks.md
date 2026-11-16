# Contract: Required checks

The `ci.yml` workflow runs on every pull request to `main`. Each job below is a required status
check in the `main` ruleset (FR-001, FR-002). Job names are the contract: renaming one requires
updating the ruleset and the README "Repository rulesets" list in the same pull request.

| Job name | Runs | Fails when |
| --- | --- | --- |
| `Lint` | `npm run lint` (ESLint + Prettier check) | Any lint or formatting error |
| `Type-check` | `npm run typecheck` (`tsc --noEmit` per workspace) | Any type error |
| `Unit tests and coverage` | `npm run test:coverage` (Node from `.nvmrc`) | Any failing test, or line coverage < 70% over all application and infrastructure TypeScript source (FR-003) |
| `API tests (Node 24)` | `npm test -w apps/api` on Node 24, the Lambda runtime (R7) | Any failing API test |
| `Build` | `npm run build` (Vite web bundle + esbuild Lambda bundles) | Any build error |
| `CDK synth` | `npm run build && npm run synth` (application app and ci-access app; no AWS credentials) | Either CDK app cannot synthesize |
| `Secret scan` | `gitleaks` over the pull request's commits | A secret is detected |

Rules:

- Required jobs have no job-level `if:` conditions and the workflow has no path filters. GitHub
  treats a skipped required job as passing, which would bypass FR-002.
- All `uses:` references are pinned to full commit SHAs.
- Every job runs locally with the same root script (US4), except `Secret scan`, which runs locally
  with `gitleaks git` when gitleaks is installed.
