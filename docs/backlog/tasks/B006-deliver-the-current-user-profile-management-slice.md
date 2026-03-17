## Title

Deliver the current-user profile management slice

## ID

B006

## Milestone

M1-foundation

## Objective

Implement the local profile read/update flow so authenticated users can view and edit app-owned profile fields.

## Background

The PRD includes basic profile management, and the auth design makes `GET /v1/me` and `PATCH /v1/me` local application concerns. This is a bounded vertical slice that proves authenticated reads and writes beyond session sync.

## Scope

- Implement `GET /v1/me` and `PATCH /v1/me`.
- Add local validation for editable fields such as display name and default timezone.
- Implement `/app/profile` in the SPA with loading, success, and error states.
- Wire the profile screen through the shared authenticated shell.

## Out of Scope

- Clerk-managed account settings such as password reset UX.
- Trip-level data.
- Avatar upload workflows unless the repo already supports them trivially.

## Dependencies

- Blockers: `B005`.
- Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`, `docs/architecture/05-auth-security.md`.

## Target Files / Areas

- `apps/api/src/modules/identity-access/`
- `apps/web/src/features/auth/` or `features/settings/`
- `apps/web/src/app/router/`

## Implementation Notes

- Keep current-user profile state in React Query or equivalent server state, not a bespoke global store.
- Limit editable fields to app-owned values defined in the architecture docs.
- Return API errors with the common envelope.

## Acceptance Criteria

- Authenticated users can fetch their local profile.
- Authenticated users can update allowed profile fields and see the changes reflected in the UI.
- Validation failures are surfaced explicitly without silent truncation or fallback behavior.

## Tests Required

- Add API tests for success, validation failure, and unauthenticated access.
- Add a frontend test covering profile form loading, success, and inline error handling.

## Observability Requirements

- Emit structured logs/traces for profile update success and failure paths.
- Do not log freeform personal content beyond IDs and allowed field names.

## Documentation Updates

- Update API docs or OpenAPI annotations for `/v1/me` endpoints if needed.

## Agent Notes

- Keep profile logic separate from trip membership and authorization logic.
- Do not let the profile page become the dumping ground for unrelated account settings.
