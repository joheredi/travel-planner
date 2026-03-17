## Title

Deliver the reservation management vertical slice

## ID

B012

## Milestone

M2-trip-planning-core

## Objective

Implement reservation CRUD with traveler associations and optional explicit itinerary linkage.

## Background

Reservations are a separate aggregate from itinerary items and need their own metadata, filters, and fast mobile access. They also become reminder targets later in the MVP.

## Scope

- Extend the schema with `Reservation` and reservation-traveler linkage.
- Implement reservation list/create/update/delete endpoints.
- Build the reservations page and forms under `/trips/:tripId/reservations`.
- Support optional linking to itinerary items and travelers while enforcing same-trip rules.

## Out of Scope

- Email import, parsing, or OCR.
- Automatic itinerary creation.
- Reminder scheduling and delivery.

## Dependencies

- Blockers: `B010`, `B011`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/reservations/`
- `apps/web/src/features/reservations/`
- `packages/contracts/` reservation DTOs

## Implementation Notes

- Keep traveler associations normalized.
- Treat itinerary linkage as optional and explicit.
- Optimize list and detail payloads for quick mobile access to provider and confirmation data.

## Acceptance Criteria

- Trip owners can create, edit, list, and delete reservations.
- Reservations can reference travelers and optionally link to itinerary items from the same trip only.
- The reservations UI is usable on mobile and surfaces key metadata clearly.

## Tests Required

- Add API tests for CRUD, validation, same-trip reference rejection, and authorization.
- Add frontend tests for reservation create/edit flows and empty/loading/error states.

## Observability Requirements

- Emit structured logs and traces for reservation mutations.
- Avoid logging confirmation codes or freeform notes in normal telemetry.

## Documentation Updates

- Update API docs and any route docs for reservation flows.

## Agent Notes

- Do not merge reservation and itinerary modules or DTOs.
- Keep provider/confirmation fields optional exactly where the PRD allows.
