## Title

Deliver the traveler management vertical slice

## ID

B010

## Milestone

M2-trip-planning-core

## Objective

Implement traveler CRUD so each trip can model the actual people traveling, including dependents and optional contact details.

## Background

Travelers are distinct from users and trip members, and reservations later depend on traveler selection. This task delivers the first trip-scoped child entity beyond the trip itself.

## Scope

- Extend the Prisma schema with `Traveler`.
- Implement traveler list/create/update/delete endpoints.
- Add the traveler management UI under the trip workspace or the trip overview/settings surface as appropriate.
- Enforce trip-scoped authorization and same-trip integrity for traveler operations.

## Out of Scope

- Reservation linkage beyond whatever is needed to keep the model ready for the later reservations task.
- Sharing flows.
- Reminder recipient logic.

## Dependencies

- Blockers: `B007`, `B008`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/travelers/`
- `apps/web/src/features/travelers/` or the chosen trip feature area
- `packages/contracts/` traveler DTOs

## Implementation Notes

- Keep travelers separate from authenticated users and memberships.
- Support dependent/minor context because the PRD calls it out explicitly.
- Make the UI mobile-friendly and lightweight; this is reference data for later flows.

## Acceptance Criteria

- Trip owners can add, edit, list, and delete travelers for a trip.
- Traveler data stays scoped to the owning trip.
- The API rejects cross-trip or unauthorized traveler access explicitly.

## Tests Required

- Add API tests for traveler CRUD success, validation failure, unauthorized access, and cross-trip rejection.
- Add frontend tests for create/edit/delete states if the UI lands in this task.

## Observability Requirements

- Log and trace traveler create/update/delete mutations with trip and user context.
- Do not log optional personal contact details beyond what is necessary for debugging IDs and outcomes.

## Documentation Updates

- Update OpenAPI docs and any trip route documentation for traveler endpoints/UI.

## Agent Notes

- Do not collapse traveler management into user account management.
- Keep field sets narrow and PRD-aligned.
