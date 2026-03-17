## Title

Deliver the notes and links vertical slice

## ID

B018

## Milestone

M3-collaboration-and-sharing

## Objective

Implement lightweight collaborative notes and important links so trips can store freeform operational context in one place.

## Background

Notes and links are explicitly in scope for the MVP and need to stay simple, trip-scoped, and mobile-friendly. They complete the shared operational hub story without requiring search or advanced content management.

## Scope

- Extend the schema with `Note` and `Link`.
- Implement CRUD endpoints for notes and links.
- Add `/trips/:tripId/notes` and `/trips/:tripId/links` screens or the route structure defined by the frontend architecture.
- Enforce collaborative role rules consistently.

## Out of Scope

- Full-text search.
- Rich document storage or attachments.
- OCR or parsing workflows.

## Dependencies

- Blockers: `B014`.
- Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/01-system-overview.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/notes-links/` or split modules
- `apps/web/src/features/notes/`
- `apps/web/src/features/links/`
- `packages/contracts/` note/link DTOs

## Implementation Notes

- Keep notes and links lightweight and editable from mobile.
- Separate notes and links at the API and route level even if the nav groups them visually.
- Do not add search infrastructure in MVP.
- Own the notes/links routes' feature-local role-aware controls here instead of treating them as part of `B015`.

## Acceptance Criteria

- Owners and editors can create, edit, and delete trip notes and links.
- Viewers can read notes and links but cannot mutate them.
- The UI surfaces notes and links cleanly on trip-scoped routes.
- The notes and links routes show feature-local read-only or disabled states consistent with owner/editor/viewer behavior.

## Tests Required

- Add API tests for notes and links CRUD plus role enforcement.
- Add frontend tests for note/link creation, empty/loading states, and viewer/editor/owner UI states.

## Observability Requirements

- Log create/update/delete events without logging note bodies or raw URLs beyond safe diagnostics.
- Preserve request correlation for these mutations.

## Documentation Updates

- Update API docs and route docs for notes and links.

## Agent Notes

- Avoid letting note bodies leak into logs or telemetry.
- Keep the UX simple and operational.
