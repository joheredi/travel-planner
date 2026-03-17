## Title

Deliver the itinerary management vertical slice

## ID

B011

## Milestone

M2-trip-planning-core

## Objective

Implement itinerary CRUD, chronological ordering, and day-by-day presentation for trip schedules.

## Background

The itinerary is one of the highest-value planning surfaces in the PRD and later serves as a reminder target. It has its own ordering rules and must stay distinct from reservations.

## Scope

- Extend the schema with `ItineraryItem`.
- Implement itinerary list/create/update/delete/reorder endpoints.
- Build the itinerary page and forms under `/trips/:tripId/itinerary`.
- Apply the date/time/category ordering rules defined in the architecture docs.

## Out of Scope

- Automatic reservation-to-itinerary creation.
- Reminder scheduling.
- Sharing-specific UI beyond what existing auth already allows.

## Dependencies

- Blockers: `B007`, `B008`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/itinerary/`
- `apps/web/src/features/itinerary/`
- `packages/contracts/` itinerary DTOs

## Implementation Notes

- Use stable sort fields such as date, start time, and explicit `sortOrder`.
- Keep itinerary items trip-scoped and category-driven.
- Make the day-by-day view readable on mobile without hidden desktop-only controls.

## Acceptance Criteria

- Trip owners can create, edit, delete, and reorder itinerary items.
- The itinerary screen displays items in stable chronological order grouped by day.
- The API enforces same-trip integrity and validation for itinerary mutations.

## Tests Required

- Add API/integration tests for CRUD, ordering, and validation.
- Add frontend tests covering loading, empty state, create/edit flow, and ordering display behavior.

## Observability Requirements

- Emit telemetry for itinerary mutations and failures.
- Include trip ID and resource ID in structured logs for create/update/delete operations.

## Documentation Updates

- Update API docs and any frontend route docs for the itinerary module.

## Agent Notes

- Keep itinerary logic separate from reservations even if UI cross-links are added later.
- Handle time fields carefully and consistently with trip timezone rules.
