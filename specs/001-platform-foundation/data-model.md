# Data Model: Platform Foundation

**Feature**: [spec.md](./spec.md) | **Date**: 2026-11-16

This entry creates no persisted application tables. The database exists empty; entries 002 and
later add the schema. The entities below are runtime and operational records.

## Health status

Returned by `GET /api/health` ([contract](./contracts/health.openapi.yaml)).

| Field | Type | Rules |
| --- | --- | --- |
| `status` | `"healthy"` \| `"unhealthy"` | `healthy` only if a `SELECT 1` against the database succeeds within the timeout (FR-006) |

- HTTP 200 with `healthy`; HTTP 503 with `unhealthy`.
- No other fields: no versions, hostnames, error text or timings (FR-006: reveals only
  healthy/unhealthy).
- The cause of an `unhealthy` result is logged, not returned.

## Error response

Returned for any unexpected backend error ([schema](./contracts/error-response.schema.json)).

| Field | Type | Rules |
| --- | --- | --- |
| `error.code` | string | `INTERNAL_ERROR` for unexpected errors; later entries add codes |
| `error.message` | string | Generic, user-safe text; never a stack trace or internal detail (FR-011) |
| `error.requestId` | string | Equals the `x-request-id` response header and the log entry's `requestId` |

## Log entry

Written by backend components to CloudWatch Logs ([schema](./contracts/log-entry.schema.json)).

| Field | Type | Rules |
| --- | --- | --- |
| `timestamp` | ISO-8601 string | Required |
| `level` | `DEBUG` \| `INFO` \| `WARN` \| `ERROR` | Required |
| `message` | string | Required |
| `service` | string | Required, e.g. `api-health` |
| `requestId` | string | Required for request-scoped entries |
| `error` | object (`name`, `message`, `stack`) | Present for `ERROR` entries only |

- Never contains secrets, credentials, database connection strings or tokens (FR-010).
- Retention: 7 days (FR-010).

## Dev environment

| Attribute | Rules |
| --- | --- |
| Source | Exactly one commit on `main`; the latest merged commit after each deploy (FR-007) |
| Lifecycle | `absent` → `deploying` → `deployed` → (`deploying` on next merge) → `destroyed` |
| Data | Disposable; destroyed with the environment; redeploy starts empty (FR-012a) |

A failed deploy rolls back to the previous `deployed` state (CloudFormation rollback).

## Required check

| Attribute | Rules |
| --- | --- |
| Name | One of the names in [required-checks.md](./contracts/required-checks.md) |
| Result | `success` \| `failure`; a missing or pending check blocks merge (FR-002) |
