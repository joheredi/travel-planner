## Title

Deliver Service Bus scheduling and worker dispatch for reminders

## ID

B022

## Milestone

M4-reminders-and-ops

## Objective

Implement reminder scheduling through Azure Service Bus and idempotent worker-side dispatch orchestration.

## Background

The async architecture depends on scheduled queue messages, stale-message tolerance, and worker-side state rehydration. This task adds that transport and execution backbone.

## Scope

- Publish scheduled reminder messages from the API when reminder state changes require it.
- Implement the `reminder-dispatch` queue contract.
- Build the worker consumer that rehydrates reminder state, rejects stale/cancelled work, and manages processing transitions.
- Configure queue-driven worker scaling inputs required for the dispatch path.

## Out of Scope

- Email payload rendering and provider-specific delivery logic.
- Dashboards and alerting beyond the instrumentation needed to expose worker behavior.
- End-user retry tooling.

## Dependencies

- Blockers: `B009`, `B021`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/06-async-reminders.md`, `docs/architecture/07-azure-deployment.md`.

## Target Files / Areas

- `apps/api/src/modules/reminders-notifications/`
- `apps/worker/src/modules/reminders-notifications/`
- `apps/worker/src/common/messaging/`
- `infra/bicep/` messaging and worker modules

## Implementation Notes

- Use queue messages as delivery triggers, not canonical state.
- Guard worker transitions so concurrent or duplicate deliveries cannot double-dispatch reminders.
- Prefer bounded platform-managed retry over custom retry chains.

## Acceptance Criteria

- Reminder creation/update/cancel publishes the expected scheduled message behavior.
- The worker can consume due messages, reload reminder state, and safely no-op stale or cancelled reminders.
- The system tolerates at-least-once delivery semantics without duplicate successful dispatch state.

## Tests Required

- Add integration tests for queue message shaping, stale-message rejection, and worker duplicate suppression paths.
- Add failure-path tests for retryable worker errors.

## Observability Requirements

- Emit traces/logs for queue publish, queue receive, processing start, stale-message no-op, retryable failure, and terminal failure paths.
- Include correlation ID, reminder ID, trip ID, queue message ID, and recipient user ID.

## Documentation Updates

- Document the queue contract and worker expectations if internal docs exist for async processing.

## Agent Notes

- Do not trust queue payloads blindly; always rehydrate from PostgreSQL.
- Keep the worker side-effect free until provider delivery is explicitly invoked.
