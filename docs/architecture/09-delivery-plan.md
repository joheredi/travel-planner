# Delivery Plan for Travel Planner MVP

## Purpose

This document turns the PRD and architecture set into a four-milestone delivery plan for the Travel Planner MVP. It keeps the sequence implementation-oriented, minimizes dependency risk, and preserves the locked architecture:

- React + TypeScript + Vite SPA
- NestJS API
- NestJS worker
- PostgreSQL with Prisma
- Clerk authentication
- Azure Static Web Apps, Azure Container Apps, Azure Service Bus, Azure Key Vault
- OpenTelemetry with Azure Monitor / Application Insights
- Bicep-managed infrastructure

## Delivery strategy

The MVP should ship as four milestones that progressively add capability on top of a stable architectural spine:

1. establish the monorepo, deployment baseline, authentication, and trip ownership model
2. deliver the core trip-planning domains in a mostly owner-driven flow
3. add collaboration, role enforcement across the planning surface, and the remaining shared planning modules
4. add async reminders plus the operational hardening needed to run the system reliably

This sequence avoids building async or collaboration complexity before the core trip data model, APIs, and UI shells are stable.

## Milestone summary

| Milestone | Goal | Primary outputs | Why it comes here |
|---|---|---|---|
| `M1-foundation` | Stand up the product and platform spine | repo bootstrap, shared packages, Prisma baseline, auth/session sync, profile, trip CRUD, trip shell, dev deployment baseline | Everything else depends on authenticated users, trip ownership, API conventions, and deployable environments |
| `M2-trip-planning-core` | Deliver the first useful single-trip planning workflow | travelers, itinerary, reservations, trip overview summary, mobile CRUD flows for owner-led planning | These are the highest-value operational entities and create the data foundations used by later collaboration and reminders |
| `M3-collaboration-and-sharing` | Turn planning into a shared trip workspace | trip sharing, owner/editor/viewer enforcement, checklists, templates, notes, links, budget, role-aware UI | Collaboration is safer once the core domains already exist and permissions can be threaded through real feature flows instead of speculative abstractions |
| `M4-reminders-and-ops` | Make the MVP operationally reliable | reminder APIs, worker execution, Service Bus integration, email delivery, notification history, dashboards, alerts, release hardening | Async delivery depends on stable reminder targets, membership rules, infrastructure, and observability foundations from earlier milestones |

## Ordering rationale

### Why foundation comes first

`M1` establishes the architectural constraints that every later milestone depends on:

- monorepo layout and shared package boundaries
- deployable `web`, `api`, and `worker` runtimes
- Prisma schema and migration workflow
- Clerk-backed session bootstrap and local user sync
- trip ownership and owner membership invariants
- API envelopes, validation, and baseline telemetry

Without these in place, later feature work would either duplicate infrastructure decisions or need rework when the platform baseline solidifies.

### Why trip-planning core comes before collaboration

`M2` focuses on the main operational data model first: travelers, itinerary items, and reservations. These domains drive the most important day-to-day trip workflows and establish the entities that later features depend on:

- reservations and itinerary items are the primary reminder targets
- travelers are needed for reservation associations
- overview summaries need real planning data before they are worth refining

Deferring collaboration until after these modules exist reduces risk. It lets the team prove CRUD flows, sorting, validation, and trip-scoped integrity in a simpler owner-led model before layering owner/editor/viewer authorization across every screen and mutation path.

### Why collaboration groups the remaining shared planning modules

`M3` is the point where the app becomes the shared operational hub described in the PRD. The milestone adds:

- owner-only sharing flows
- owner/editor/viewer enforcement across existing modules
- checklists and checklist templates
- notes and important links
- budget tracking

These features are naturally collaboration-heavy and benefit from a stabilized access model. They also complete the trip-scoped content surface before reminder automation is introduced.

### Why reminders and ops are last

`M4` intentionally lands after the content model is complete. Reminder delivery depends on:

- existing itinerary, reservation, and checklist item records
- stable trip membership and recipient validation
- working API and worker runtimes
- Service Bus, Key Vault, email delivery, and telemetry setup

Putting reminders earlier would pull queue timing, duplicate suppression, failure handling, and operator workflows into a codebase that had not yet settled its domain behavior. By landing reminders last, the team can build them against stable entities and finish with production-minded release gates, dashboards, and failure visibility.

## Critical path work

The following work sits on the critical path for the MVP:

1. **Platform and repo foundation**
   - `pnpm` + Turborepo workspace setup
   - `apps/web`, `apps/api`, `apps/worker`
   - `packages/db`, `packages/contracts`, `packages/ui`
   - local development and CI entrypoints

2. **Persistence and contract baseline**
   - Prisma schema and migrations for user, trip, membership, and planning entities
   - shared enums and request/response contracts
   - API validation and error envelope conventions

3. **Identity and authorization spine**
   - Clerk integration in the SPA
   - `POST /v1/session/sync`
   - local `User` bootstrap
   - trip ownership and `TripMember` invariants
   - role-aware authorization checks in the API

4. **Deployable environment baseline**
   - Bicep modules for core Azure resources
   - GitHub Actions CI/CD with OIDC
   - dev environment deployment for SPA, API, and worker
   - Key Vault and managed identity wiring

5. **Reminder infrastructure**
   - Service Bus queue and worker consumption path
   - email delivery adapter
   - idempotent dispatch and dead-letter handling
   - observability and release checks

If any of these streams slip, downstream milestone work will either block or accumulate rework.

## Cross-milestone rules

- Keep the MVP a modular monolith even though it deploys as `web`, `api`, and `worker`.
- Preserve `TripMember` as the only source of trip authorization.
- Keep travelers separate from authenticated users.
- Keep reservations and itinerary items as separate modules.
- Keep reminder delivery asynchronous through Service Bus and the worker.
- Keep production secrets in Key Vault and infrastructure in Bicep.
- Add observability and tests in every milestone; do not defer all quality work to the end.

## Milestone documents

- `docs/backlog/milestones/M1-foundation.md`
- `docs/backlog/milestones/M2-trip-planning-core.md`
- `docs/backlog/milestones/M3-collaboration-and-sharing.md`
- `docs/backlog/milestones/M4-reminders-and-ops.md`

Each milestone document defines its objective, scope boundaries, dependencies, completion criteria, and main delivery risks.
