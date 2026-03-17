## Title

Deliver release gates, rollout workflow, and operational runbooks

## ID

B025

## Milestone

M4-reminders-and-ops

## Objective

Finish the MVP delivery system with staging/prod promotion rules, rollout ordering, smoke checks, and operator-facing runbook guidance.

## Background

The deployment and quality docs require approval-gated promotion, ordered rollouts, migration safety, and restore awareness. This task converts those expectations into concrete release mechanics and documentation.

## Scope

- Extend GitHub Actions or deployment automation for staging and production gates.
- Encode or document the required rollout order: infra, migrations, API, worker, frontend.
- Add smoke checks for API, worker, and migration-sensitive release paths.
- Write restore/rollback/runbook guidance needed for MVP operations.

## Out of Scope

- Multi-region disaster recovery.
- Per-feature canary infrastructure beyond simple revision-based rollback.
- Enterprise change-management workflows.

## Dependencies

- Blockers: `B009`, `B023`, `B024`.
- Architecture refs: `docs/architecture/07-azure-deployment.md`, `docs/architecture/08-observability-testing.md`.

## Target Files / Areas

- `.github/workflows/`
- `infra/bicep/` if staging/prod infra definitions need completion
- operator or release docs under `docs/`
- smoke-test scripts or workflow steps

## Implementation Notes

- Favor explicit approval gates and immutable artifacts.
- Keep runbooks practical: deploy, validate, rollback, restore, and investigate reminder failures.
- Make smoke checks focused on the critical user journeys and async health, not exhaustive E2E coverage.

## Acceptance Criteria

- Staging and production promotion paths exist with approval gates.
- Release automation or runbooks document the required rollout order and rollback expectations.
- Smoke checks cover API health, worker health, migration success, and at least the critical user paths needed for release confidence.

## Tests Required

- Validate workflow syntax and deployment steps.
- Exercise smoke checks in a non-production environment before considering the task done.

## Observability Requirements

- Ensure release markers or equivalent deployment metadata can be tied to dashboards and alerts.
- Keep rollout failures observable through existing logs, traces, and workflow output.

## Documentation Updates

- Create or update release and operational runbooks covering deploy, rollback, restore, and reminder-failure investigation.

## Agent Notes

- This task should make the product operable, not just deployable.
- Keep the process boring, explicit, and Azure-aligned.
