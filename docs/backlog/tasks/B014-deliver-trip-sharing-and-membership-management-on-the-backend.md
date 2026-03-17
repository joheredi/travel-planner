## Title

Deliver trip sharing and membership management on the backend

## ID

B014

## Milestone

M3-collaboration-and-sharing

## Objective

Implement backend support for listing trip members, adding collaborators, updating roles, and revoking access for existing users.

## Background

The MVP collaboration model is intentionally narrow: owners share trips with existing accounts, and the API remains the source of authorization decisions. This task establishes that sharing backbone.

## Scope

- Implement trip-member list, add, update, and revoke endpoints.
- Enforce owner-only membership changes and role invariants.
- Normalize email lookups and support idempotent share-by-email behavior.
- Return collaboration-safe member DTOs only.

## Out of Scope

- Pending invitations for users without accounts.
- Frontend sharing screens.
- Full authorization threading across all existing modules beyond the shared backend policy foundation needed by later tasks.

## Dependencies

- Blockers: `B005`, `B007`.
- Architecture refs: `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`.

## Target Files / Areas

- `apps/api/src/modules/identity-access/`
- `apps/api/src/modules/trip-management/` or dedicated membership module
- `packages/contracts/` member DTOs

## Implementation Notes

- Preserve the single-owner invariant; do not allow adding a second owner.
- Sharing must work only with existing local user accounts.
- Return controlled `USER_NOT_FOUND` behavior inside the authorized sharing workflow.

## Acceptance Criteria

- Owners can list members, add editors/viewers by email, change roles, and revoke access.
- Editors and viewers cannot mutate membership state.
- The API prevents invalid ownership transitions and duplicate membership rows.

## Tests Required

- Add API tests for owner/editor/viewer/non-member behavior.
- Cover email normalization, idempotent add behavior, and invalid owner-role transitions.

## Observability Requirements

- Log and trace member add, re-role, and revoke actions as security-relevant events.
- Include trip ID, actor user ID, target member ID, and request correlation.

## Documentation Updates

- Update OpenAPI docs for trip-member endpoints and any security documentation references if needed.

## Agent Notes

- Keep membership logic centralized so later feature modules can reuse one policy source.
- Do not expose Clerk internals in member responses.
