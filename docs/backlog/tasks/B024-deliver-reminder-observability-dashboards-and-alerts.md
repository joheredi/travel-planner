## Title

Deliver reminder observability dashboards and alerts

## ID

B024

## Milestone

M4-reminders-and-ops

## Objective

Add the production-facing observability needed to operate API requests, worker processing, and reminder delivery safely.

## Background

The quality and async architecture docs treat logs, traces, metrics, dashboards, and alerts as part of the feature. Reminders are too operationally risky to ship without that visibility.

## Scope

- Validate and close any gaps in the reminder instrumentation added by `B021`-`B023`, including required spans, logs, and metrics.
- Create or define Application Insights dashboards for API health, worker health, reminder delivery, and dependency health.
- Create or define alerts for 5xx rate, latency, dead-letter count, backlog age, and provider failure spikes.
- Ensure correlation data survives from API request through queue processing and provider calls.

## Out of Scope

- Broader BI or product analytics not tied to operational health.
- Multi-region or advanced SRE tooling beyond the MVP stack.
- Frontend-heavy telemetry beyond the light baseline already described in architecture.

## Dependencies

- Blockers: `B009`, `B022`, `B023`.
- Architecture refs: `docs/architecture/06-async-reminders.md`, `docs/architecture/07-azure-deployment.md`, `docs/architecture/08-observability-testing.md`.

## Target Files / Areas

- `apps/api/src/common/telemetry/`
- `apps/worker/src/common/telemetry/`
- `infra/bicep/` observability modules
- runbook or ops docs if created in-repo

## Implementation Notes

- Treat instrumentation as a feature concern owned primarily by `B021`-`B023`; this task validates that coverage and adds any missing glue needed for dashboards and alerts.
- Model dashboards and alerts as delivery artifacts, not later ops work.
- Keep logs structured and privacy-safe.
- Use environment-scoped dashboards and alert definitions aligned with the Azure deployment doc.

## Acceptance Criteria

- Reminder API and worker paths emit the traces, logs, and metrics required by the architecture docs.
- Dashboards exist for API health, worker/reminder health, and dependency health.
- Alerts exist for key failure categories such as dead letters, delivery failures, and sustained API degradation.

## Tests Required

- Add automated tests for any instrumentation helpers that contain business logic.
- Validate alert/dashboard configuration or generated infra definitions as part of CI where practical.

## Observability Requirements

- This task is itself the observability implementation.
- Ensure correlation fields include request ID, correlation ID, reminder ID, trip ID, queue message ID, and recipient user ID where relevant.

## Documentation Updates

- Document dashboard ownership, alert intent, and key correlation fields in developer/operator docs.

## Agent Notes

- Do not settle for raw console logs; use the structured telemetry standards from the architecture docs.
- Keep privacy rules intact even in non-prod diagnostics.
