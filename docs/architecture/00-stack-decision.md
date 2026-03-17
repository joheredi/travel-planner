# Stack Decision for Travel Planner MVP

## Executive Summary

Travel Planner will ship as a TypeScript monorepo built with `pnpm` workspaces and `Turborepo`.

- Frontend: React + TypeScript + Vite
- Backend API: NestJS + TypeScript
- Background processing: a separate NestJS worker application for reminders and notifications
- Database: PostgreSQL with Prisma
- Authentication: Clerk as the managed identity provider
- Frontend hosting: Azure Static Web Apps
- Backend hosting: Azure Container Apps
- Async messaging: Azure Service Bus
- Secrets: Azure Key Vault
- Observability: OpenTelemetry with Azure Monitor / Application Insights
- Infrastructure as code: Bicep

This stack optimizes for a small-team MVP that needs strong CRUD, clear access control, mobile-friendly performance, and reliable asynchronous reminders without taking on unnecessary platform complexity.

## Why This Stack Fits the PRD

The PRD describes a collaborative planning tool with structured entities such as trips, travelers, itinerary items, reservations, checklists, budget entries, reminders, and roles. That shape is strongly relational, not document-first, so PostgreSQL is a better fit than a schemaless store.

React with Vite fits the requirement for a responsive web app optimized for mobile while keeping the frontend simple. We do not need server-rendering complexity or a full-stack React framework to deliver the MVP user flows.

NestJS keeps the backend in TypeScript, which lowers cognitive load inside a monorepo and makes contracts, validation, and shared types easier to manage. It also gives a clean home for server-side authorization, audit-friendly mutations, and reminder orchestration.

Azure Container Apps and Azure Service Bus fit the MVP reminder requirement well. Request/response traffic stays in the API, while reminder scheduling and notification delivery run asynchronously and can be retried without blocking users.

Clerk removes a large amount of non-differentiated auth work. The MVP needs sign-up, sign-in, password reset, and collaborator access, but it does not win by building credential storage, token issuance, or account recovery in-house.

The Azure-aligned hosting and operations choices keep deployment and observability straightforward while avoiding Kubernetes-level overhead that the MVP does not need.

## Locked Decisions

The following decisions are immutable for the MVP unless a later architecture document explicitly supersedes them:

- Use a single `pnpm` + `Turborepo` monorepo.
- Build the frontend as a React + TypeScript + Vite SPA.
- Build the backend as NestJS services in TypeScript.
- Use PostgreSQL as the system of record.
- Use Prisma for schema management, migrations, and database access.
- Use Clerk for authentication; do not build custom credential flows.
- Host the SPA on Azure Static Web Apps.
- Host API and worker containers on Azure Container Apps.
- Use Azure Service Bus for asynchronous reminder and notification workflows.
- Store secrets in Azure Key Vault and inject them into runtime environments.
- Use OpenTelemetry and Azure Monitor / Application Insights for logs, traces, and metrics.
- Provision cloud infrastructure with Bicep.
- Keep the MVP as a modular monolith, not a microservices fleet.
- Prefer REST + JSON HTTP APIs for the MVP.
- Optimize for mobile-responsive web usage, not native apps or offline-first behavior.

## Chosen Stack in More Detail

### Frontend

The frontend will be a React SPA built with Vite and TypeScript. It will call the NestJS API over HTTPS using Clerk-issued bearer tokens.

Key frontend principles:

- route-level code splitting for primary trip areas
- mobile-first responsive layouts
- React Query for server state and caching
- form validation close to the UI boundary
- no direct database or service-bus access from the client

### Backend API

The backend will be a NestJS REST API responsible for:

- authentication token verification
- trip-scoped authorization
- CRUD for all MVP entities
- reminder scheduling requests
- audit-friendly logging for critical mutations

The API remains the only writer of business state from user-facing flows.

### Background Processing

Reminder and notification execution will run asynchronously through Azure Service Bus. Background processing lives in `apps/worker`, a separate NestJS application deployed as its own Azure Container App while still remaining part of the same modular monolith and codebase.

This keeps queue consumption, retries, and delivery side effects out of synchronous API request handling.

### Data Layer

PostgreSQL is the canonical store for trips, memberships, travelers, itinerary items, reservations, checklists, notes, links, budget entries, reminders, and notifications.

Prisma is the required access layer for:

- schema definitions
- migrations
- generated client access
- local and CI schema consistency

Raw SQL is allowed only for narrowly justified performance or migration cases.

### Authentication and Authorization

Authentication is delegated to Clerk.

- Clerk manages sign-up, sign-in, password reset, and session/token flows.
- The frontend uses Clerk SDKs.
- The API verifies Clerk JWTs and maps authenticated users to application records.
- Trip roles such as Owner, Editor, and Viewer remain application-level authorization concerns enforced server-side.

This separation keeps identity management outsourced while preserving domain-specific permission logic in the backend.

## Explicitly Rejected Alternatives

### Next.js Full-Stack

Rejected because the MVP does not need server-rendered React, server actions, or a full-stack framework to satisfy the PRD. A Vite SPA plus NestJS API keeps concerns clearer, reduces framework overlap, and fits the locked hosting split of Azure Static Web Apps plus Azure Container Apps.

### .NET Backend

Rejected because it would introduce a second primary language and duplicate platform expertise in a small MVP team. NestJS keeps frontend and backend on TypeScript, which improves shared contracts, onboarding speed, and agent-friendly execution inside one monorepo.

### MongoDB / Azure DocumentDB

Rejected because the MVP data model is relational and permission-sensitive. Trips, members, travelers, itinerary items, reservations, budgets, reminders, and role assignments all benefit from relational integrity, transactions, and explicit querying. PostgreSQL fits that model better and keeps reporting simpler.

### Azure App Service Instead of Azure Container Apps

Rejected because Container Apps better matches the need for containerized API and worker workloads, scale-to-zero-friendly non-prod setups, and event-driven background processing. App Service would work, but it is a worse fit for the split API/worker runtime pattern we expect for reminders.

### Self-Hosted Auth

Rejected because authentication is commodity infrastructure with real security and lifecycle costs. Building password storage, password reset, session hardening, email verification, and account recovery would slow the MVP and create unnecessary operational risk. Managed auth is the better trade-off.

### Kubernetes / AKS

Rejected because it adds major operational surface area without solving an MVP problem. The product needs reliable deployment, background jobs, observability, and secrets management, not cluster administration. Azure Container Apps provides enough control with far less overhead.

## Monorepo Conventions

### Package Manager

- Use `pnpm` workspaces only.
- Do not mix `npm`, `yarn`, or `bun` lockfiles into the repository.
- The root lockfile is authoritative in CI and local development.

### Task Runner

- Use `Turborepo` for build, lint, test, type-check, and dev task orchestration.
- Cache deterministic tasks locally and in CI.
- Define tasks per app/package rather than using ad hoc shell scripts where normal package scripts are sufficient.

### Apps and Packages Layout

The default repository layout is:

```text
apps/
  web/        React + Vite frontend
  api/        NestJS HTTP API
  worker/     NestJS-based background worker for reminders and notifications
packages/
  db/         Prisma schema, migrations, generated client wrapper
  contracts/  Shared DTOs, API contracts, domain enums, validation schemas
  ui/         Shared React UI primitives used by the web app
  eslint-config/
  tsconfig/
```

Rules:

- Put business logic in `apps/api` or shared backend-safe packages, not in the frontend.
- Put database schema and migrations in `packages/db`.
- Put cross-app contract types in `packages/contracts`.
- Only create a new package when at least two apps will use it or it isolates a true boundary.
- Avoid speculative packages such as generic `core`, `shared-utils`, or `platform` buckets without a concrete reason.

### Environment Variable Strategy

- Keep runtime env files app-scoped, not repo-scoped.
- Commit `apps/web/.env.example`, `apps/api/.env.example`, and `apps/worker/.env.example`.
- Use the `VITE_` prefix only for variables intentionally exposed to the browser.
- Validate server env vars at startup.
- Pull production secrets from Azure Key Vault through deployment configuration, not from checked-in files.
- Never read server secrets in the frontend build.

### Linting, Formatting, and Testing Defaults

- Linting: ESLint
- Formatting: Prettier
- Frontend unit/component tests: Vitest + React Testing Library
- Backend tests: Jest + Supertest
- End-to-end tests: Playwright for critical user journeys only
- Type checking: `tsc --noEmit`

Defaults:

- every app and package must type-check
- every user-facing feature requires automated tests at the appropriate level
- mobile viewport behavior must be validated for frontend work
- API changes must include request validation and authorization coverage

## Deployment Topology

At a high level, the MVP deploys as:

1. Azure Static Web Apps hosts the React frontend.
2. The frontend calls the NestJS API hosted on Azure Container Apps.
3. The API persists relational data in Azure Database for PostgreSQL.
4. The API publishes reminder and notification work to Azure Service Bus.
5. A worker process on Azure Container Apps consumes Service Bus messages and performs reminder scheduling or delivery actions.
6. Secrets are sourced from Azure Key Vault.
7. Telemetry flows through OpenTelemetry into Azure Monitor / Application Insights.

Principles:

- keep synchronous request handling in the API
- keep retries and delivery side effects in worker consumers
- keep infrastructure declarative in Bicep
- keep environments separated at the Azure resource level

## Local Development Standards

- Standard runtime: Node.js 22 LTS and `pnpm`
- Run PostgreSQL locally with Docker Compose or an equivalent containerized workflow.
- Run `web`, `api`, and `worker` through Turborepo dev tasks.
- Use seeded local data for realistic trip, traveler, itinerary, checklist, and budget scenarios.
- Prefer local development without cloud dependencies except where Azure resources are intrinsic, such as Service Bus integration testing.
- When a cloud-backed integration is required, use a dedicated shared dev environment provisioned by Bicep rather than ad hoc personal resources.
- Keep reminder delivery adapters non-destructive in local development by default.
- Do not rely on manual database edits; use Prisma migrations and seed scripts.

## Definition of Done for Implementation Tasks

An implementation task is not done unless all of the following are true:

- code follows the locked stack and monorepo conventions in this document
- new or changed data structures are reflected in Prisma schema and migrations when needed
- request validation, authorization, and error handling are implemented explicitly
- relevant automated tests are added or updated
- logs, traces, and metrics are added for important mutations or async processing
- mobile-responsive behavior is considered for frontend work
- environment variables and secrets are documented through `.env.example` or deployment config changes
- Bicep is updated when infrastructure requirements change
- the feature is verifiable end to end in local or shared dev
- documentation is updated when the change affects developer workflow or architecture assumptions

## Non-Goals and Anti-Patterns

### Non-Goals

- building a booking marketplace
- adding native mobile apps
- adopting offline-first architecture
- introducing advanced AI itinerary generation into the MVP core
- designing for enterprise-scale multi-tenancy
- splitting the MVP into independent microservices

### Anti-Patterns

- putting domain rules in React components
- bypassing server-side authorization because the UI hides an action
- storing secrets in source control or plain `.env` files outside approved local use
- adding new infrastructure without Bicep changes
- using Mongo-style denormalization for relational trip data
- handling reminder delivery inline during user requests
- introducing Kubernetes, CQRS, event sourcing, or GraphQL without a concrete MVP need
- creating shared packages before real reuse exists
- letting downstream tasks reopen locked stack choices without an explicit architecture decision record

## Decision Outcome

This stack gives the Travel Planner MVP a clear, low-ambiguity foundation:

- one language across frontend and backend
- strong support for structured collaborative data
- a simple but production-ready Azure deployment model
- a managed auth path that avoids reinventing security plumbing
- a clean route to asynchronous reminders and notifications

Downstream architecture, API, backlog, and delivery documents should treat this decision as the baseline.
