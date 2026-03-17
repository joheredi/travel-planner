## Title

Deliver email reminder rendering and notification history

## ID

B023

## Milestone

M4-reminders-and-ops

## Objective

Complete the reminder delivery path by sending email reminders through Azure Communication Services Email and recording notification outcomes.

## Background

Email is the only required delivery channel in the MVP. Once queue scheduling and worker execution exist, the system still needs provider integration and end-user visible delivery history.

## Scope

- Integrate the worker with Azure Communication Services Email.
- Render reminder email payloads from current trip and target data.
- Persist successful, failed, and dead-letter-adjacent notification outcomes.
- Ensure notification history APIs surface meaningful delivery state to authorized users.

## Out of Scope

- Additional delivery channels such as SMS or push.
- A full in-app notification center.
- User-managed resend/retry UI.

## Dependencies

- Blockers: `B021`, `B022`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/06-async-reminders.md`, `docs/architecture/07-azure-deployment.md`.

## Target Files / Areas

- `apps/worker/src/modules/reminders-notifications/`
- `apps/worker/src/common/` email adapters
- `apps/api/src/modules/reminders-notifications/` if notification read models need adjustment
- `infra/bicep/communications`

## Implementation Notes

- Render emails from fresh database state, not denormalized queue payloads.
- Store enough provider metadata to troubleshoot failures without over-logging user content.
- Keep notification history read-only from user-facing APIs.

## Acceptance Criteria

- Due reminders can send email successfully through the configured provider path.
- Notification records reflect success and failure outcomes accurately.
- Authorized users can view reminder/notification history for their trips.

## Tests Required

- Add worker integration tests covering successful send, transient failure, and terminal failure recording.
- Add API tests for notification-history reads and authorization.

## Observability Requirements

- Emit provider-call spans, delivery outcome logs, retry counters, and failure metrics.
- Avoid logging full reminder content, note bodies, or sensitive reservation details in provider telemetry.

## Documentation Updates

- Document required email provider configuration and any safe non-destructive local/testing mode.

## Agent Notes

- Keep the provider integration replaceable but do not over-abstract it.
- Preserve idempotency across retry and redelivery paths.
