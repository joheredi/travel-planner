## Title

Deliver the trips shell and owner trip management UI

## ID

B008

## Milestone

M1-foundation

## Objective

Build the authenticated trip routes and owner-facing trip management flows on top of the trip API.

## Background

The frontend architecture centers the app around `/trips` and `/trips/:tripId/*`. This task turns the backend trip slice into a usable owner workflow in the SPA.

## Scope

- Implement `/trips`, `/trips/new`, `/trips/:tripId/overview`, and `/trips/:tripId/settings`.
- Create the authenticated shell and trip shell structure needed for those routes.
- Implement create, edit, archive/delete, and trip navigation flows for owner users.
- Handle loading, empty, success, and error states across the trip routes.

## Out of Scope

- Traveler, itinerary, reservation, or collaboration screens.
- Final overview summary panels for later milestones.
- Role-aware affordances beyond the owner-led path needed in `M1`.

## Dependencies

- Blockers: `B005`, `B007`.
- Architecture refs: `docs/architecture/04-frontend-architecture.md`, `docs/architecture/03-api-design.md`, `docs/prd/travel-planner-prd.md`.

## Target Files / Areas

- `apps/web/src/app/shells/`
- `apps/web/src/app/router/`
- `apps/web/src/features/trips/`
- `apps/web/src/features/overview/`
- `packages/ui/` if reusable presentational components are needed

## Implementation Notes

- Keep route-level data loading and feature-local components aligned with the frontend architecture.
- Use the shared API client and React Query patterns, not ad hoc fetch calls.
- Treat frontend role gating as affordance only; backend auth remains authoritative.

## Acceptance Criteria

- Users can create and manage trips entirely through the SPA.
- The app redirects correctly between public, authenticated, and trip-scoped routes.
- The core trip routes behave well on mobile-sized viewports.

## Tests Required

- Add page/component tests for trips list, create trip form, and trip settings states.
- Include routing and mutation-success/error coverage.

## Observability Requirements

- Ensure request correlation headers flow through the shared API client.
- Capture client-side route errors through the app error boundary if frontend telemetry is already wired.

## Documentation Updates

- Update route maps or frontend docs if they are maintained in-repo.

## Agent Notes

- Do not overdesign the overview route in `M1`; it can start as a lightweight shell.
- Prefer reusable form patterns that later trip-scoped modules can copy.
