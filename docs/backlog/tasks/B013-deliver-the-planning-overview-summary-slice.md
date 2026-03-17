## Title

Deliver the planning overview summary slice

## ID

B013

## Milestone

M2-trip-planning-core

## Objective

Upgrade the trip overview page so it summarizes the planning data created in `M2` rather than acting as a placeholder shell.

## Background

The frontend architecture expects the trip overview to be the summary landing page for a selected trip. Once travelers, itinerary items, and reservations exist, the overview can become operationally useful.

## Scope

- Add overview summary queries or response fields for traveler counts, upcoming itinerary items, and upcoming reservations.
- Implement those panels on `/trips/:tripId/overview`.
- Make overview navigation into itinerary and reservations clear on mobile.
- Keep the aggregation logic owned by appropriate backend modules rather than stuffing it into the UI.

## Out of Scope

- Checklist, notes, links, or budget summary panels.
- Sharing management UI.
- Reminder surfacing.

## Dependencies

- Blockers: `B010`, `B011`, `B012`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/04-frontend-architecture.md`, `docs/prd/travel-planner-prd.md`.

## Target Files / Areas

- `apps/api/src/modules/trip-management/`
- `apps/web/src/features/overview/`
- `packages/contracts/` if overview DTOs need extension

## Implementation Notes

- Prefer lightweight summary payloads over huge nested aggregates.
- Keep panel counts and upcoming lists consistent with the same sort rules used on detail pages.
- Treat the overview as navigation-friendly and mobile-first.

## Acceptance Criteria

- The overview page shows traveler counts, upcoming itinerary items, and upcoming reservations from real data.
- The page loads with proper loading, empty, and error states.
- Users can navigate from overview panels into the relevant planning routes.

## Tests Required

- Add backend tests for overview summary shaping if new query/service logic is introduced.
- Add frontend tests covering overview summary rendering and empty states.

## Observability Requirements

- Rely on baseline API request telemetry; add summary-query traces if the aggregation logic becomes non-trivial.

## Documentation Updates

- Update overview route docs or API docs if new summary payloads are introduced.

## Agent Notes

- Do not pre-add panels for later modules just to reserve space.
- Keep overview latency reasonable for mobile use.
