## Title

Create the Prisma database package and core persistence schema

## ID

B003

## Milestone

M1-foundation

## Objective

Stand up `packages/db` with Prisma, the initial database schema, and migration flow for the core identity and trip ownership model.

## Background

The MVP uses PostgreSQL with Prisma as the only system of record. Later feature slices depend on a working `User`, `Trip`, and `TripMember` schema plus a reliable migration workflow.

## Scope

- Initialize Prisma in `packages/db`.
- Model `User`, `Trip`, and `TripMember` with the invariants described in the domain model.
- Add generated client access wiring suitable for API and worker consumption.
- Add the first migration and a minimal seed strategy if needed for local development.

## Out of Scope

- Traveler, itinerary, reservation, checklist, budget, reminder, or notification tables.
- Application feature logic outside the persistence package.
- Cloud database provisioning.

## Dependencies

- Blockers: `B001`, `B002`.
- Architecture refs: `docs/architecture/02-domain-model.md`, `docs/architecture/07-azure-deployment.md`, `docs/architecture/08-observability-testing.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `packages/db/prisma/migrations/`
- `packages/db/src/`
- local database bootstrap files such as `docker-compose.yml` if needed

## Implementation Notes

- Preserve the invariant that every trip has one owner and one matching owner membership.
- Use UUID primary keys and explicit timestamps.
- Keep schema names aligned with the conceptual model so later tasks do not have to rename tables or fields.

## Acceptance Criteria

- Prisma can generate a client successfully.
- The initial migration applies cleanly to a local PostgreSQL instance.
- The schema supports local user records, owned trips, and trip memberships without violating the owner invariants.

## Tests Required

- Run Prisma generate and migration commands successfully.
- Add integration coverage or migration validation proving owner membership invariants hold on trip creation paths once the API task consumes the schema.

## Observability Requirements

- No end-user telemetry yet.
- Leave database access structured so later API and worker tasks can instrument Prisma calls cleanly.

## Documentation Updates

- Document local database bootstrap and migration commands if those instructions are introduced in the repo.

## Agent Notes

- Do not pull future entities into the first schema just because they are known.
- Optimize for stable names and clean ownership boundaries.
