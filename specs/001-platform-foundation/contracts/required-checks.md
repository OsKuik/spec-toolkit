# Contract: Required checks

The `ci.yml` workflow runs on every pull request to `main`. Each job below is a required status
check in the `main` ruleset (FR-001, FR-002). Job names are the contract: renaming one requires
updating the ruleset and the README "Repository rulesets" list in the same pull request.

| Job name | Runs | Fails when |
| --- | --- | --- |
| `Lint` | `npm run lint` (ESLint + Prettier check) | Any lint or formatting error |
| `Type-check` | `npm run typecheck` (`tsc --noEmit` per workspace) | Any type error |
| `Unit tests and coverage` | `npm run test:coverage` | Any failing test, or combined line coverage < 70% (FR-003) |
| `Build` | `npm run build` (web bundle + Lambda bundles) | Any build error |
| `CDK synth` | `npm run synth` | The CDK app cannot synthesize |
| `Secret scan` | `gitleaks` over the pull request's commits | A secret is detected |

Every job runs locally with the same root script (US4), except `Secret scan`, which runs locally
with `gitleaks git` when gitleaks is installed.
