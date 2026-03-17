## Title

Deliver the Clerk session sync authentication slice

## ID

B005

## Milestone

M1-foundation

## Objective

Implement Clerk-backed authentication across the SPA and API, including session bootstrap and local user sync.

## Background

Identity is delegated to Clerk, but the app still needs a local `User` record and a trusted auth boundary. This task wires the SPA auth shell to the API session-sync path described in the architecture docs.

## Scope

- Integrate Clerk into the React app root and public/authenticated shells.
- Implement backend Clerk token verification and authenticated request context.
- Implement `POST /v1/session/sync`.
- Wire frontend bootstrap so authenticated sessions call session sync and store the returned user context in server state.

## Out of Scope

- Editable local profile fields beyond what is needed to sync the user record.
- Trip features beyond proving the auth boundary works.
- Sharing or membership management UI.

## Dependencies

- Blockers: `B003`, `B004`.
- Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/05-auth-security.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `apps/web/src/app/providers/`
- `apps/web/src/app/router/`
- `apps/web/src/lib/api/auth.ts`
- `apps/api/src/common/auth/`
- `apps/api/src/modules/identity-access/`

## Implementation Notes

- Use Clerk as the identity source and local `User` records as the application identity mapping.
- Differentiate `401` and `403` handling cleanly in the frontend client.
- Do not build custom login or logout endpoints in the API.

## Acceptance Criteria

- A signed-in user can authenticate through Clerk and complete `POST /v1/session/sync`.
- Authenticated API requests receive a trusted local user context.
- Expired or invalid tokens are rejected with explicit `401` responses.
- The SPA protects authenticated routes through the shared auth boundary.

## Tests Required

- Add API tests for missing, invalid, and valid bearer tokens on session sync.
- Add frontend tests or screen tests for authenticated bootstrap and unauthenticated redirect behavior.

## Observability Requirements

- Log and trace session sync attempts, user creation/upsert outcomes, and auth failures without exposing tokens.
- Attach request correlation to the auth flow.

## Documentation Updates

- Document required Clerk environment variables in the relevant `.env.example` files.

## Agent Notes

- Normalize email during local user creation.
- Do not trust frontend role or membership data as auth input.
