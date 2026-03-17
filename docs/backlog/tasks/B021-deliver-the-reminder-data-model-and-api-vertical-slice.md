## Title

Deliver the reminder data model and API vertical slice

## ID

B021

## Milestone

M4-reminders-and-ops

## Objective

Implement reminder and notification persistence plus the reminder CRUD API surface without worker-side delivery yet.

## Background

Reminders are the main async feature in the MVP. The API owns user intent, reminder validation, and durable state, while actual delivery is deferred to the worker.

## Scope

- Extend the schema with `Reminder` and `Notification` plus the async-safe fields called out in the reminders architecture doc.
- Implement reminder create/update/cancel/list endpoints and notification-history read endpoints.
- Validate recipients, target ownership, and same-trip invariants.
- Add idempotency support for reminder creation.

## Out of Scope

- Service Bus scheduling and worker processing.
- Actual email delivery.
- In-app notifications or recurring reminder logic.

## Dependencies

- Blockers: `B011`, `B012`, `B014`, `B016`.
- Architecture refs: `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`, `docs/architecture/06-async-reminders.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/reminders-notifications/`
- `packages/contracts/` reminder DTOs

## Implementation Notes

- Support all three MVP reminder targets in this task: itinerary items, reservations, and checklist items.
- Model exactly one target foreign key per reminder and one recipient per reminder row.
- Add schedule-version and lifecycle fields needed for safe async execution.
- Keep delivery history read-only to end users.

## Acceptance Criteria

- Owners and editors can create, update, cancel, and list reminders for itinerary items, reservations, and checklist items.
- Viewers can read reminder and notification history but cannot mutate reminder state.
- The API enforces target ownership, recipient membership, idempotency rules, and same-trip validation explicitly.
- Invalid reminder schedules are rejected explicitly instead of creating unusable reminder rows.

## Tests Required

- Add API tests for create/update/cancel/list flows, role enforcement, cross-trip rejection, and idempotency behavior.
- Add schema/integration tests for reminder lifecycle constraints.

## Observability Requirements

- Log reminder create/update/cancel operations with correlation, trip ID, reminder ID, and recipient user ID.
- Avoid logging freeform reminder content beyond IDs and safe status fields.

## Documentation Updates

- Update OpenAPI docs for reminder and notification endpoints.
- Update any persistence docs if new async-safe fields are added.

## Agent Notes

- Keep the API responsible for durable user intent only.
- Do not send email inline in this task.
