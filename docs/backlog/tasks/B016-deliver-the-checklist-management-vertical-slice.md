## Title

Deliver the checklist management vertical slice

## ID

B016

## Milestone

M3-collaboration-and-sharing

## Objective

Implement trip checklists and checklist item completion so preparation work can be tracked collaboratively.

## Background

Checklists are a core MVP capability and later become reminder targets. The domain model treats checklists as trip-scoped content with item-level completion state.

## Scope

- Extend the schema with `Checklist` and `ChecklistItem`.
- Implement checklist and checklist-item CRUD endpoints.
- Implement the `/trips/:tripId/checklists` UI with quick completion toggles.
- Enforce owner/editor/viewer permissions for checklist operations.

## Out of Scope

- Checklist templates.
- Reminder scheduling.
- Advanced checklist automation or recurring lists.

## Dependencies

- Blockers: `B014`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/checklists/`
- `apps/web/src/features/checklists/`
- `packages/contracts/` checklist DTOs

## Implementation Notes

- Track completion at the checklist-item level, not through redundant aggregate flags.
- Keep the UI optimized for quick toggles on mobile.
- Use trip-scoped ownership and role checks consistently.
- Own the checklist route's feature-local role-aware controls here instead of waiting on broader UI polish from `B015`.

## Acceptance Criteria

- Owners and editors can create, edit, delete, and complete checklist items.
- Viewers can read checklist state but cannot mutate it.
- Checklist state persists correctly per trip and per checklist.
- The checklist route shows feature-local read-only or disabled states consistent with owner/editor/viewer behavior.

## Tests Required

- Add API tests for CRUD, completion toggles, role enforcement, and cross-trip rejection.
- Add frontend tests for checklist loading, create flow, quick-complete behavior, and viewer/editor/owner UI states.

## Observability Requirements

- Emit logs/traces for checklist creation, item completion, and destructive actions.
- Include trip ID and checklist/checklist-item IDs in mutation logs.

## Documentation Updates

- Update API docs and route docs for the checklist module.

## Agent Notes

- Keep checklist behavior simple and operational.
- Do not prebuild reminder logic into checklist persistence yet.
