## Title

Deliver the checklist template vertical slice

## ID

B017

## Milestone

M3-collaboration-and-sharing

## Objective

Implement reusable checklist templates owned by a user and copied into trips on demand.

## Background

The PRD and domain model call for reusable checklist templates, but templates remain user-owned and copied into trips rather than live-linked. This is a separate bounded slice from ordinary trip checklists.

## Scope

- Extend the schema with `ChecklistTemplate` and template items.
- Implement template CRUD endpoints and apply-to-trip behavior.
- Add the template selection/creation flows needed by the checklist UI.
- Keep template ownership user-scoped and trip instantiation explicit.

## Out of Scope

- Global/shared template marketplaces.
- Template versioning or retroactive sync into existing trip checklists.
- Reminder automation.

## Dependencies

- Blockers: `B006`, `B016`.
- Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/checklists/`
- `apps/web/src/features/checklists/`
- `packages/contracts/` template DTOs

## Implementation Notes

- Template application should copy items into trip-scoped checklists, not create live links.
- Keep template CRUD separated from trip checklist CRUD even if the UI groups them.
- User ownership comes from the authenticated user, not trip membership.

## Acceptance Criteria

- Users can create, edit, delete, and list their own checklist templates.
- Users can instantiate a trip checklist from a template.
- Editing a template does not retroactively change already-created trip checklists.

## Tests Required

- Add API tests for template CRUD and apply-to-trip behavior.
- Add frontend tests for template creation and template-application flows.

## Observability Requirements

- Log template create/update/delete and template-application actions with user and trip context where relevant.

## Documentation Updates

- Update checklist docs or API docs to explain template ownership and copy semantics.

## Agent Notes

- Do not turn templates into trip-scoped records.
- Keep the MVP UX focused on practical reuse, not taxonomy-heavy management.
