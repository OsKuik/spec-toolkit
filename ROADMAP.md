# Roadmap

Plan-only ledger for the Kanban Board MVP. Format: [docs/roadmap-format.md](docs/roadmap-format.md).
Progress lives in GitHub issues and pull requests, not here.

## Planned Specs

### 001 — Platform foundation  [status: planned]

- **Description:** AWS CDK (TypeScript) application, a reproducible production-like AWS dev
  environment, CI running lint, type-check, unit tests, build and CDK synth, and a logging and
  error-handling baseline.
- **Outcome:** Every later feature can be built, checked in CI and deployed to the AWS dev
  environment from the repository.
- **Scope (in):** CDK app skeleton, dev environment stacks, CI required checks, logging baseline.
- **Scope (out):** Everything listed as out of scope in constitution Principle I.
- **Depends on:** none
- **Governed by:** Constitution v1.0.0 (Principles V, VI, VII, IX)
- **Notes:** The architecture proposal, including the serverless decision, needs full-team review
  before implementation (Principle III).

### 002 — Authentication and workspaces  [status: planned]

- **Description:** Sign-in through AWS Cognito and a minimal workspace model with workspace
  membership.
- **Outcome:** Users can sign in and belong to a workspace that owns boards.
- **Scope (in):** Cognito sign-up and sign-in, workspace creation, workspace membership.
- **Scope (out):** Everything listed as out of scope in constitution Principle I.
- **Depends on:** 001
- **Governed by:** Constitution v1.0.0 (Principles I, IV)
- **Notes:** Security-sensitive; needs documented security review before deployment.

### 003 — Basic permissions  [status: planned]

- **Description:** Server-side authorization for workspace, board and issue actions.
- **Outcome:** Users can only see and change data their workspace role allows.
- **Scope (in):** Basic roles and server-side permission checks with automated tests.
- **Scope (out):** Everything listed as out of scope in constitution Principle I.
- **Depends on:** 002
- **Governed by:** Constitution v1.0.0 (Principles IV, VII)
- **Notes:** Security-sensitive; needs documented security review before deployment.

### 004 — Boards and columns  [status: planned]

- **Description:** Create boards with columns within a workspace.
- **Outcome:** Workspace members can create a board and see its columns.
- **Scope (in):** Board creation, column definition, board view.
- **Scope (out):** Everything listed as out of scope in constitution Principle I.
- **Depends on:** 003
- **Governed by:** Constitution v1.0.0 (Principle I)

### 005 — Issue CRUD  [status: planned]

- **Description:** Create, read, update and delete issues on a board.
- **Outcome:** Users can manage issues on a board within their permissions.
- **Scope (in):** Issue create, read, update and delete with automated tests.
- **Scope (out):** Everything listed as out of scope in constitution Principle I.
- **Depends on:** 004
- **Governed by:** Constitution v1.0.0 (Principles I, VII)

### 006 — Drag-and-drop and ordering  [status: planned]

- **Description:** Move issues between columns by drag-and-drop and reorder issues within a column,
  with the order persisted.
- **Outcome:** Board state reflects where users drop issues, and the order survives reloads.
- **Scope (in):** Drag-and-drop between columns, ordering within a column, persistence, automated
  tests for ordering.
- **Scope (out):** Everything listed as out of scope in constitution Principle I.
- **Depends on:** 005
- **Governed by:** Constitution v1.0.0 (Principles I, VII)

### 007 — Comments  [status: planned]

- **Description:** Add and view comments on issues.
- **Outcome:** Users can discuss an issue in its comments.
- **Scope (in):** Adding comments and listing an issue's comments.
- **Scope (out):** Everything listed as out of scope in constitution Principle I.
- **Depends on:** 005
- **Governed by:** Constitution v1.0.0 (Principle I)
