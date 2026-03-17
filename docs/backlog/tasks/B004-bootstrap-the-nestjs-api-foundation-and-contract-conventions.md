## Title

Bootstrap the NestJS API foundation and contract conventions

## ID

B004

## Milestone

M1-foundation

## Objective

Create the API bootstrap, validation, error handling, OpenAPI generation, and base telemetry hooks that all later backend tasks will reuse.

## Background

The API design and security docs require stable `/v1` routing, request validation, response envelopes, structured error envelopes, and OpenAPI generation. Those cross-cutting concerns should be implemented once, not feature by feature.

## Scope

- Initialize `apps/api` as a NestJS application.
- Configure global validation, common exception mapping, and response envelope conventions.
- Establish the first shared API contract primitives in `packages/contracts` for response envelopes, common errors, and the baseline auth/profile/trip DTO surface that later slices can extend.
- Set up code-first OpenAPI generation and a versioned spec artifact or endpoint.
- Add shared API modules for auth context, errors, telemetry, and validation helpers.

## Out of Scope

- Feature-specific endpoints beyond any minimal health/bootstrap routes.
- Clerk-specific session flows.
- Feature-specific trip, traveler, reminder, or sharing business logic.
- Worker runtime implementation.

## Dependencies

- Blockers: `B001`, `B002`, `B003`.
- Architecture refs: `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`, `docs/architecture/08-observability-testing.md`.

## Target Files / Areas

- `apps/api/src/main.ts`
- `apps/api/src/common/auth/`
- `apps/api/src/common/errors/`
- `apps/api/src/common/telemetry/`
- `apps/api/src/common/validation/`
- `packages/contracts/`

## Implementation Notes

- Return stable error codes and statuses matching the API design.
- Make DTO classes the source of truth for OpenAPI generation.
- Put shared envelope and common error contract types in `packages/contracts` so later slices extend one baseline instead of inventing parallel shapes.
- Keep the envelope and exception logic reusable so later controllers stay thin.

## Acceptance Criteria

- The API boots locally.
- Global validation rejects malformed requests with the expected error envelope.
- Baseline envelope and common error contract types exist in `packages/contracts` for downstream slices to reuse.
- OpenAPI generation is wired and produces a versioned spec artifact or endpoint from the running application bootstrap.
- Common API modules exist for downstream feature controllers and services.

## Tests Required

- Add API-level tests for validation failure and error-envelope behavior.
- Verify the generated OpenAPI document is emitted successfully as part of the API build or validation flow.
- Run build and type-check for `apps/api` successfully.

## Observability Requirements

- Add baseline request logging and OpenTelemetry bootstrap hooks for the API runtime.
- Include request ID or correlation ID plumbing in the common layer.

## Documentation Updates

- Document API bootstrap and contract-generation expectations if developer docs exist.

## Agent Notes

- Do not hide validation or auth failures behind generic 500 responses.
- Keep common modules boring and easy for later agents to reuse.
