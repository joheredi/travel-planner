## Title

Deliver the trip CRUD and owner membership backend slice

## ID

B007

## Milestone

M1-foundation

## Objective

Implement the backend trip model, CRUD endpoints, and owner membership enforcement for the owner-led MVP path.

## Background

Trips are the top-level aggregate for the product, and later traveler, itinerary, sharing, checklist, and reminder tasks all depend on a stable trip API plus owner membership invariants.

## Scope

- Implement `GET /v1/trips`, `POST /v1/trips`, `GET /v1/trips/:tripId`, `PATCH /v1/trips/:tripId`, and `DELETE /v1/trips/:tripId`.
- Create owner membership automatically on trip creation.
- Enforce owner-only deletion and owner/editor update rules that already apply in the base model.
- Return trip detail payloads compatible with the frontend route structure.

## Out of Scope

- Sharing other than the implicit owner membership.
- Traveler, itinerary, reservation, checklist, budget, or reminder child resources.
- Overview aggregation beyond counts that naturally belong in trip detail.

## Dependencies

- Blockers: `B003`, `B004`, `B005`.
- Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`.

## Target Files / Areas

- `apps/api/src/modules/trip-management/`
- `packages/contracts/` for trip DTOs/enums
- `packages/db/` if minor schema extensions are required

## Implementation Notes

- Preserve the owner-membership invariant on create and destructive actions.
- Use explicit trip status handling rather than UI-only derived state.
- Return `404` and `403` behavior consistent with the API design for inaccessible resources.

## Acceptance Criteria

- Authenticated users can create, list, view, update, archive, and delete their own trips through the API.
- Trip creation persists both the trip and the matching owner membership.
- Only authorized callers can mutate or delete trip records.

## Tests Required

- Add API integration tests for create, list, detail, update, and delete flows.
- Cover owner-only deletion and validation/not-found cases.

## Observability Requirements

- Emit structured logs/traces for trip creation, update, archival, and deletion.
- Include trip ID, authenticated user ID, and request correlation in important mutation logs.

## Documentation Updates

- Update OpenAPI annotations and any backend module docs for trip endpoints.

## Agent Notes

- Do not pull sharing flows into this task.
- Keep detail responses small enough for mobile while still useful for route bootstrap.
