## Title

Deliver the sharing UI and role-aware frontend affordances

## ID

B015

## Milestone

M3-collaboration-and-sharing

## Objective

Build the sharing screen and add role-aware affordances across the trip UI so collaborative users can work safely.

## Background

Once backend membership management exists, the frontend needs a dedicated sharing route and clear role-aware presentation. The frontend architecture explicitly separates sharing and settings because permissions differ.

## Scope

- Implement `/trips/:tripId/sharing`.
- Add member list, add collaborator, update role, and revoke flows.
- Apply common role-aware action visibility and disabled states across the existing M1/M2 trip routes and shell affordances.
- Handle `401` and `403` UI outcomes distinctly.

## Out of Scope

- Server-side authorization logic itself.
- Pending-invite UX.
- Real-time presence or collaborative cursors.
- Feature-local role-aware controls for checklist, notes/links, and budget routes introduced later in `M3`; those belong to `B016`, `B018`, and `B019`.

## Dependencies

- Blockers: `B014`, `B008`, `B013`.
- Architecture refs: `docs/architecture/04-frontend-architecture.md`, `docs/architecture/05-auth-security.md`, `docs/prd/travel-planner-prd.md`.

## Target Files / Areas

- `apps/web/src/features/sharing/`
- `apps/web/src/app/shells/TripShell.tsx`
- role-aware UI components in trip feature folders

## Implementation Notes

- Treat role-aware UI as affordance only; do not duplicate business authorization rules in complex client logic.
- Make sharing flows work on mobile-sized viewports.
- Keep member forms and error messages explicit and low-friction.
- Keep this task focused on the sharing route plus common shell-level affordances; later feature slices own their own route-specific role-aware controls.

## Acceptance Criteria

- Owners can manage collaborators from the sharing route.
- Editors and viewers cannot use the sharing route to mutate membership and see appropriate read-only or denied states there.
- Existing M1/M2 trip routes show owner/editor/viewer-appropriate shell and action affordances without claiming ownership of later M3 feature-specific controls.
- Frontend behavior distinguishes authentication problems from authorization problems.

## Tests Required

- Add frontend tests for sharing flows and role-aware rendering on the sharing route and existing M1/M2 trip routes.
- Add regression tests proving viewers cannot access mutation affordances.

## Observability Requirements

- Ensure sharing mutations carry request correlation through the shared API client.
- Capture frontend error-boundary behavior for sharing route failures if frontend telemetry exists.

## Documentation Updates

- Update route docs or UX notes for the sharing screen if maintained in-repo.

## Agent Notes

- Do not hide the sharing route behind brittle client-only assumptions; rely on backend responses too.
- Keep forms focused on email and role only.
