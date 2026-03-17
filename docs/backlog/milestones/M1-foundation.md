# M1 - Foundation

## Objective

Stand up the deployable product spine for Travel Planner so later feature milestones can build on stable identity, trip ownership, persistence, API conventions, and frontend shells instead of inventing them feature by feature.

## In-scope capabilities

- monorepo bootstrap for `apps/web`, `apps/api`, `apps/worker`, `packages/db`, `packages/contracts`, and `packages/ui`
- baseline developer workflow with `pnpm`, Turborepo, lint, type-check, test, and build scripts
- Prisma schema and migrations for:
  - `User`
  - `Trip`
  - `TripMember`
  - shared enums and foundational timestamps/IDs
- Clerk integration in the SPA and backend token verification in the API
- `POST /v1/session/sync`, `GET /v1/me`, and `PATCH /v1/me`
- trip list, create, detail, update, archive/delete flows for the owner path
- authenticated frontend shell, trips list route, trip settings route, and baseline trip overview shell
- API conventions from the architecture docs:
  - versioned `/v1` routes
  - response envelopes
  - structured error envelopes
  - request validation
- baseline OpenTelemetry, structured logging, and request correlation for web-facing API flows
- Azure dev deployment baseline:
  - Static Web App
  - API Container App
  - worker Container App scaffold
  - PostgreSQL
  - Key Vault
  - Application Insights / Log Analytics
  - GitHub Actions CI/CD with OIDC and Bicep deployment entrypoints

## Implementation slices

1. repo and package layout
2. shared contracts, config validation, and API baseline
3. Clerk bootstrap plus local user sync
4. trip persistence and owner membership invariants
5. trips UI shell and owner CRUD
6. dev environment deployment and CI checks

## Out-of-scope items

- trip sharing and non-owner role management UI
- traveler management
- itinerary, reservations, checklists, notes, links, and budget modules
- reminder creation or worker-side business logic
- in-app notification center
- private networking, WAF, Front Door, or other non-MVP Azure expansions

## Dependencies

- locked stack and repo conventions from `docs/architecture/00-stack-decision.md`
- system and runtime boundaries from `docs/architecture/01-system-overview.md`
- trip and membership invariants from `docs/architecture/02-domain-model.md`
- API conventions from `docs/architecture/03-api-design.md`
- auth/security rules from `docs/architecture/05-auth-security.md`
- Azure deployment and CI/CD model from `docs/architecture/07-azure-deployment.md`
- observability and testing expectations from `docs/architecture/08-observability-testing.md`

## Hard foundation readiness for downstream milestones

`M2` and later milestones should assume all of the following are already true before they start:

- `packages/contracts` contains the baseline response envelopes, common error shapes, and the first shared DTO surface needed by auth, profile, and trip flows
- Prisma migrations and local database bootstrap work reliably for `User`, `Trip`, and `TripMember`
- Clerk session sync and `/v1/me` flows work end to end
- trip CRUD works through the API and SPA, including owner membership creation and the architecture-required trip fields such as timezone and currency
- API validation, error envelopes, OpenAPI generation, baseline logging, and request correlation are in place
- CI runs install, lint, type-check, tests, build, Prisma validation, and Bicep validation
- the shared `dev` environment can deploy the SPA and API, with the worker present as a healthy scaffold and Application Insights / Log Analytics ready for later telemetry

## Completion criteria

- the repository boots locally with `web`, `api`, and `worker` running through Turborepo
- Clerk-authenticated users can complete session sync and manage their local profile
- an authenticated user can create, list, view, update, archive, and delete their own trips
- trip creation persists both the `Trip` and the matching owner `TripMember` record
- unauthorized requests fail with explicit `401`, `403`, `404`, `409`, or `422` behavior consistent with the API design
- the frontend has working authenticated routing for `/trips`, `/trips/new`, `/trips/:tripId/overview`, and `/trips/:tripId/settings`
- CI runs install, lint, type-check, tests, build, Prisma validation, and Bicep validation
- the `dev` Azure environment can deploy the SPA and API successfully, with the worker deployed as a healthy scaffold

## Risk notes

- The main risk is overbuilding platform scaffolding before user-visible value appears. Keep the foundation limited to the pieces later milestones truly depend on.
- Clerk integration and local user bootstrap can create drift if token verification, email normalization, and `User` upsert rules are not settled early.
- Trip detail and overview pages can become a dumping ground for future module logic. Keep `M1` overview intentionally light so later milestones can add real summary panels from actual data.
