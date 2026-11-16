<!--
Sync Impact Report
- Version change: (none) → 1.0.0 (initial ratification)
- Modified principles: none (initial adoption)
- Added principles:
  I. Narrow, Explicit Scope
  II. Spec-Driven Development
  III. Team-Based Decisions
  IV. Security-First
  V. Technology Baseline
  VI. Real Deployment Target
  VII. Quality Gate
  VIII. Git Workflow
  IX. Simplicity and Operational Clarity
  X. Traceable Work Tracking
- Added sections: Merge and Release Gates; Development Workflow; Governance
- Removed sections: none
- Templates: none modified (dependent templates read the constitution at runtime)
- Follow-up TODOs:
  - project-context.toml `[review] required_approvals` is 0; Principle VIII requires 1.
    Set it to 1 and rerun `python3 scripts/bootstrap.py github --apply`.
  - project-context.toml `[project] description` is still a placeholder.
-->

# Kanban Board Constitution

## Core Principles

### I. Narrow, Explicit Scope

Version 1 (the MVP) covers only:

- board creation;
- issue create, read, update and delete;
- drag-and-drop of issues between columns;
- ordering of issues within a column;
- comments on issues;
- a minimal authentication and workspace model;
- basic permissions.

The following are out of scope and MUST NOT appear in any spec, plan or task without a prior
constitution amendment: roadmap planning, analytics and reporting, workflow automation, enterprise
governance, large external integrations, and portfolio management. Reviewers MUST reject changes
that implement out-of-scope capabilities.

Rationale: an explicit boundary keeps the MVP deliverable and makes scope creep a visible,
governed decision rather than an accident.

### II. Spec-Driven Development

- No implementation starts without an approved `spec.md` and `plan.md` (with research, data model,
  contracts and quickstart) merged to `main`, and a generated and analyzed `tasks.md`.
- `spec.md` and `plan.md` are approved in their own pull request. The initial `tasks.md` is then
  generated and analyzed on the Phase 1 implementation branch; there is no separate tasks-only issue
  or pull request.
- `tasks.md` MAY evolve in reviewed implementation pull requests; structural edits (Phases, tasks,
  dependencies) MUST be followed by another `speckit-analyze` run.
- Every code change MUST trace to a spec requirement through its task.
- AI-generated work that did not follow this process (ungoverned work) MUST be reviewed, quarantined
  or corrected to match an approved spec before it is accepted.

Rationale: traceability from requirement to code is what lets a team, rather than an individual,
own and review the product.

### III. Team-Based Decisions

- There is no tech lead and no single release approver. Technical decisions are made by the team
  as a group.
- Architecture proposals, including any decision to adopt or reject a serverless design, MUST be
  written down and reviewed by the full team before implementation starts.
- Release sign-off MUST include explicit sign-off from every team member.
- Silence is not approval: only an explicit, recorded approval counts.

Rationale: shared ownership avoids single points of failure and keeps decisions visible.

### IV. Security-First

- Changes touching authentication, permissions, data access, secrets or IAM are security-sensitive
  and MUST have a documented security review before deployment.
- Authentication MUST use AWS Cognito.
- Authorization MUST be enforced server-side; client-side checks are for user experience only.
- Secrets MUST NOT be committed to the repository; they are held in an AWS-managed secret store.
- IAM roles and policies MUST follow least privilege.

Rationale: an issue tracker holds team data; access control failures are the most damaging defect
class for this product.

### V. Technology Baseline

- Full-stack TypeScript.
- React for the frontend.
- PostgreSQL on Amazon RDS for persistence.
- AWS Cognito for authentication.
- AWS-managed services for backend and hosting.
- AWS serverless (Lambda, event-driven processing) is preferred where practical. This preference
  MUST NOT override security, cost, reviewability or operational clarity; deviating from it is an
  architecture decision under Principle III.

Introducing a language, framework or managed service outside this baseline requires a team-reviewed
architecture proposal.

Rationale: a single, shared stack keeps every team member able to review every change.

### VI. Real Deployment Target

- Work MUST be validated in a real, production-like AWS dev environment, not only locally.
- All infrastructure MUST be defined as code with AWS CDK (TypeScript) and be reproducible from the
  repository; manual console changes to managed resources are not allowed.

Rationale: behavior that only works locally is not done; reproducible infrastructure makes the dev
environment trustworthy.

### VII. Quality Gate

- Core behavior MUST have automated tests: issue CRUD, ordering (between and within columns) and
  permissions.
- Test coverage MUST be at least 70%.
- When a spec or this constitution requires tests, each behavior change gets a test task written
  before its implementation task.
- Required checks before merge: lint, type-check, unit tests, build and CDK synth. All MUST pass.

Rationale: the core behaviors are where regressions hurt users most; automated gates keep quality
independent of who reviews.

### VIII. Git Workflow

- Work happens on short-lived, issue-backed feature branches only.
- Every change reaches `main` through a pull request; direct pushes to `main` are not allowed.
- Each pull request MUST have at least 1 approval from a reviewer other than its author
  (`required_approvals = 1`), and all required checks MUST pass before merge.
- Merges follow group discussion of the change; disagreement is resolved before merge, not after.

Rationale: independent review and protected `main` make every change visible and reversible.

### IX. Simplicity and Operational Clarity

- Prefer the simplest design that meets the spec; added complexity MUST be justified in the plan.
- Changes MUST be reviewable in small increments.
- Code MUST include logging and error handling sufficient to operate and debug the dev environment.

Rationale: a small team can only review and operate what it can understand.

### X. Traceable Work Tracking

- All work is tracked as GitHub Issues on the team's GitHub Projects Kanban board.
- Each roadmap entry has an Epic and a spec-and-plan Planning issue. Issues are derived from
  approved spec tasks at Phase granularity: each Phase in `tasks.md` is one GitHub issue, and
  individual `T###` tasks never become issues.
- Every pull request MUST link to an issue. No work starts without a linked issue.
- Scope changes are made in the spec first and then reflected in issues.

Rationale: the board is the single, shared view of what is planned, in progress and done.

## Merge and Release Gates

A pull request MAY merge only when all of the following hold:

1. It links to its issue (Principle X) and traces to an approved spec (Principle II).
2. All required checks pass: lint, type-check, unit tests, build and CDK synth (Principle VII).
3. The exact final diff has had a `code-review-and-quality` pass through the `code-reviewer` agent,
   with Critical and Required findings fixed and other findings shown to the team.
4. Security-sensitive changes have a documented security review (Principle IV).
5. It has at least 1 approval from a reviewer other than the author (Principle VIII).

A release MAY ship only when it has been validated in the AWS dev environment (Principle VI) and
every team member has explicitly signed off (Principle III).

## Development Workflow

- Spec Kit steps run in order through the `speckit-*` skills with their hooks: specify → clarify →
  plan → tasks → analyze → implement.
- The spec is reviewed and approved after clarify; the spec and plan are approved through their own
  pull request before tasks are generated.
- Architecture proposals are written in the plan (or a linked document) and reviewed by the full
  team before implementation.
- The AWS dev environment is the shared place where changes are deployed and tried before release.

## Governance

- This constitution supersedes other project practices. Where `project-context.toml`, agent
  instructions or documentation disagree with it, the constitution wins and the other source is
  corrected.
- Amendments require team-wide review and documented approval from every team member, recorded on
  the amendment's pull request.
- Versioning follows semantic versioning:
  - MAJOR: removal or backward-incompatible redefinition of a principle or governance rule.
  - MINOR: a new principle or section, or materially expanded guidance.
  - PATCH: clarifications, wording and typo fixes with no semantic change.
- Every pull request review checks compliance with this constitution. Scope additions listed as out
  of scope in Principle I require an amendment before any spec, plan or task includes them.

**Version**: 1.0.0 | **Ratified**: 2026-11-16 | **Last Amended**: 2026-11-16
