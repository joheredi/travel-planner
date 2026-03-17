# System Overview for Travel Planner MVP

## Purpose

This document defines the high-level system architecture for the Travel Planner MVP. It builds on `docs/architecture/00-stack-decision.md` and describes the runtime shape of the application, the major bounded contexts, and the constraints that downstream design documents must respect.

The system is a modular monolith deployed as multiple runtimes:

- a React SPA for the user-facing product
- a NestJS API for synchronous business operations
- a NestJS worker for asynchronous reminder and notification processing

## System Context

### Primary Users

- household trip planners organizing family travel
- couples coordinating shared trip plans
- small-group coordinators sharing trip details with other travelers
- invited collaborators with editor or viewer access

### External Systems and Services

- Clerk for authentication, session management, and password flows
- Azure Static Web Apps for hosting the SPA
- Azure Container Apps for hosting API and worker runtimes
- Azure Database for PostgreSQL as the managed relational database
- Azure Service Bus for reminder and notification messaging
- Azure Key Vault for secret storage and runtime secret injection
- Azure Monitor / Application Insights for traces, logs, and metrics
- Azure Communication Services Email for MVP reminder delivery

### System Context Summary

Travel Planner sits between end users and a small set of managed platform services. Users interact only with the React SPA. The SPA authenticates through Clerk and calls the NestJS API. The API owns all synchronous domain behavior and persists data in PostgreSQL. Reminder execution is delegated to Service Bus and the worker runtime, with operational visibility flowing into Application Insights.

## ASCII Component Diagram

```text
               +----------------------+
               |   Primary Users      |
               | families / couples / |
               | small-group planners |
               +----------+-----------+
                          |
                          v
               +----------------------+
               | React SPA            |
               | apps/web             |
               | Azure Static Web App |
               +---+--------------+---+
                   |              |
     auth flows    |              | HTTPS JSON API
                   v              v
          +----------------+   +----------------------+
          | Clerk          |   | NestJS API           |
          | managed auth   |   | apps/api             |
          +----------------+   +---+----+----+---+---+
                                   |    |    |   |
                                   |    |    |   +--> Application Insights
                                   |    |    |
                                   |    |    +------> Azure Key Vault
                                   |    |
                                   |    +-----------> Azure Service Bus
                                   |
                                   +-----------> PostgreSQL
                                                        ^
                                                        |
                                         +--------------+------+
                                         | NestJS Worker       |
                                         | apps/worker         |
                                         | reminder processing |
                                         +------+----------+---+
                                                |          |
                                                |          +--> Application Insights
                                                |
                                                +--> Azure Communication Services Email
```

## Major Runtime Components

### React SPA

The React SPA in `apps/web` is the only user-facing runtime.

Responsibilities:

- render all authenticated and unauthenticated product screens
- manage client-side routing and feature navigation
- acquire user identity and tokens through Clerk
- call the API for all domain reads and writes
- provide responsive mobile-first UX

Non-responsibilities:

- direct database access
- authorization decisions beyond UI affordances
- reminder execution
- secret access

### NestJS API

The NestJS API in `apps/api` is the synchronous backend boundary.

Responsibilities:

- verify authenticated requests
- enforce trip-scoped authorization
- execute CRUD and business rules for all MVP modules
- persist state in PostgreSQL through Prisma
- schedule reminder work by publishing to Service Bus
- emit telemetry for requests and critical mutations

Non-responsibilities:

- long-running background processing
- direct rendering of UI
- storing auth credentials

### PostgreSQL

PostgreSQL is the system of record for application data.

Responsibilities:

- store all canonical trip and collaboration data
- preserve relational integrity between users, trips, members, travelers, itinerary items, reservations, checklists, notes, budget entries, reminders, and notification records
- support transactional writes for multi-entity mutations

Non-responsibilities:

- scheduling background jobs on its own
- serving as a message queue

### Reminder Worker

The worker in `apps/worker` is the asynchronous execution boundary.

Responsibilities:

- consume due reminder messages from Service Bus
- rehydrate reminder state from PostgreSQL
- execute reminder delivery logic
- record delivery outcomes and avoid duplicate sends
- emit telemetry for message handling and failures

Non-responsibilities:

- handling interactive user requests
- owning domain writes unrelated to reminder execution

### Azure Service Bus

Azure Service Bus is the delivery backbone for reminder work.

Responsibilities:

- hold reminder and notification messages until they are ready for consumption
- decouple reminder scheduling from reminder processing
- support retries and dead-letter handling

### Azure Key Vault

Key Vault provides secret management for:

- Clerk backend secrets
- database credentials
- email delivery credentials
- any other runtime-only sensitive configuration

### Application Insights

Application Insights is the primary sink for:

- API request telemetry
- worker message telemetry
- exception tracking
- reminder delivery monitoring
- dependency tracing for PostgreSQL, Service Bus, Clerk, and email delivery

## High-Level Request Flows

### 1. Sign In

1. The user opens the SPA.
2. The SPA redirects to or embeds Clerk authentication UI.
3. Clerk authenticates the user and issues a session/token.
4. The SPA includes the Clerk bearer token on API requests.
5. The API verifies the token, resolves the application user record, and returns authorized data.

Design note:

- Identity comes from Clerk.
- application permissions still come from Travel Planner domain data

### 2. Create Trip

1. An authenticated user submits trip details in the SPA.
2. The SPA sends the request to the API.
3. The API validates input, verifies the user, and creates the trip plus owner membership in PostgreSQL.
4. The API returns the created trip payload.
5. The SPA updates local state and routes the user to the trip overview.

### 3. Share Trip

1. The trip owner opens sharing settings in the SPA.
2. The owner enters the collaborator email and selects a role.
3. The SPA sends the share request to the API.
4. The API verifies the requester is an owner and looks up an existing application user linked to Clerk identity.
5. If the collaborator exists, the API creates or updates the `TripMember` record in PostgreSQL.
6. The SPA reflects the new membership list.

MVP simplification:

- collaborators must already have accounts
- there are no pending invitations in the MVP

### 4. Create Reminder

1. A user creates a reminder on a checklist item, reservation, or itinerary item.
2. The SPA sends the request to the API.
3. The API validates the request, checks authorization, and creates the reminder record in PostgreSQL.
4. The API publishes a scheduled message to Azure Service Bus that references the reminder.
5. The API returns success to the SPA.

### 5. Process Reminder

1. A scheduled reminder message becomes available in Azure Service Bus.
2. The worker consumes the message.
3. The worker loads the reminder, related trip context, and delivery state from PostgreSQL.
4. The worker checks idempotency rules to avoid duplicate notification delivery.
5. The worker sends the reminder email through Azure Communication Services Email.
6. The worker persists the delivery outcome and emits telemetry.
7. Failed deliveries are retried through Service Bus policies and eventually dead-lettered for operator review if needed.

## Bounded Contexts and Modules

The MVP should be implemented as bounded modules inside a modular monolith. Each module owns its business rules, service layer, persistence mapping, and API endpoints.

### Identity and Access

Owns:

- application user profile
- Clerk identity mapping
- trip memberships
- Owner/Editor/Viewer role enforcement

Does not own:

- trip content
- traveler records
- reminder execution

### Trip Management

Owns:

- trip lifecycle
- trip metadata
- trip status
- trip overview aggregation roots

Does not own:

- itinerary ordering rules
- reservation details
- checklist completion logic

### Itinerary

Owns:

- itinerary items
- day-by-day ordering
- itinerary categories and schedule fields

Does not own:

- booking confirmation records
- trip membership rules

### Reservations

Owns:

- reservation records
- provider and confirmation metadata
- reservation-specific dates, links, and traveler associations

Does not own:

- itinerary rendering rules
- reminder delivery

MVP simplification:

- reservations do not automatically create itinerary items
- linking between reservations and itinerary items is optional, not implicit

### Checklists

Owns:

- trip checklists
- checklist items
- completion state
- checklist template instantiation

Does not own:

- reminder delivery
- trip sharing rules

### Notes and Links

Owns:

- freeform notes
- important links
- basic ordering and editing behavior

Does not own:

- full-text search infrastructure for MVP

### Budget

Owns:

- budget entries
- estimate versus actual amounts
- trip-level totals by category

Does not own:

- advanced accounting rules
- currency conversion engines

### Reminders and Notifications

Owns:

- reminder configuration
- scheduling requests
- delivery attempts
- notification history

Does not own:

- the canonical state of checklist items, reservations, or itinerary items

MVP simplification:

- email is the only required delivery channel
- an in-app notification center is explicitly deferred unless it is nearly free

## Responsibility Boundaries by Runtime

### `apps/web`

- UI composition
- local interaction state
- authenticated navigation
- API client integration

### `apps/api`

- synchronous domain commands and queries
- authorization enforcement
- persistence orchestration
- message publishing

### `apps/worker`

- message consumption
- reminder execution
- retry-safe side effects

### `packages/db`

- Prisma schema
- migrations
- generated client wrapper and shared DB access utilities

### `packages/contracts`

- shared request and response DTOs
- validation schemas that can be reused at boundaries
- shared enums for cross-runtime consistency

### `packages/ui`

- reusable presentational components only
- no backend-aware business logic

## Recommended Repo Structure

```text
apps/
  web/
    src/
      app/
      routes/
      features/
        auth/
        trips/
        itinerary/
        reservations/
        checklists/
        notes-links/
        budget/
        reminders/
      components/
      lib/
  api/
    src/
      main.ts
      modules/
        identity-access/
        trip-management/
        itinerary/
        reservations/
        checklists/
        notes-links/
        budget/
        reminders-notifications/
      common/
        auth/
        errors/
        telemetry/
        validation/
  worker/
    src/
      main.ts
      modules/
        reminders-notifications/
      common/
        telemetry/
        messaging/
packages/
  db/
    prisma/
  contracts/
  ui/
  eslint-config/
  tsconfig/
infra/
  bicep/
```

Repo rules:

- keep module names consistent between frontend features and backend modules where possible
- do not create a separate service or package for every entity
- keep shared packages small and intentional
- prefer module-local code over generic utility dumping grounds

## Cross-Cutting Concerns

### Authentication and Authorization

- Clerk is the source of identity.
- The API is the source of authorization decisions.
- Every trip-scoped mutation must enforce membership and role checks server-side.
- Travelers are separate from users and trip members because some travelers are dependents, not account holders.

### Validation

- Validate all external input at API boundaries.
- Reuse shared contracts where it reduces drift, but keep domain invariants enforced in backend services.
- Reject invalid or unauthorized writes explicitly rather than silently ignoring them.

### Error Handling

- Return structured API errors with stable error codes where practical.
- Distinguish validation failures, authorization failures, not-found cases, and dependency failures.
- Do not hide background delivery failures; record them and surface them through telemetry and operator workflows.

### Telemetry

- Instrument API requests, key mutations, database dependencies, Service Bus operations, and email delivery attempts.
- Include correlation identifiers so a user action can be traced from SPA request through API and worker processing.
- Treat reminder processing, sharing changes, and authorization failures as first-class telemetry events.

### Background Jobs

- Background work must flow through Azure Service Bus and `apps/worker`.
- Do not execute reminder delivery inline in request handlers.
- Use idempotent worker logic so retries do not create duplicate notifications.
- Dead-lettered messages are operational failures, not silent drops.

## Risks and Simplifications for MVP

### Risks

- sharing and role enforcement can become inconsistent if authorization logic leaks out of the identity-access boundary
- reminder processing can generate duplicate or late notifications if idempotency and retry behavior are underdesigned
- frontend screens can become hard to evolve if trip overview pages aggregate too much module-specific logic directly in the UI
- module boundaries can erode if every feature reaches into shared tables without clear ownership

### Simplifications

- no real-time collaborative editing; users see changes through normal API refresh flows
- collaborators must already have accounts
- travelers remain distinct from users and trip members
- reservations remain separate from itinerary items
- email is the only required notification channel in MVP
- the system remains a modular monolith even though API and worker run as separate deployables

## Constraints for Downstream Design Docs

- Treat `docs/architecture/00-stack-decision.md` as immutable unless a later decision record explicitly replaces it.
- Preserve the modular-monolith architecture; do not split bounded contexts into separate services for the MVP.
- Keep the three runtime apps: `web`, `api`, and `worker`.
- Model travelers separately from authenticated users and trip memberships.
- Keep sharing server-enforced and role-based with `Owner`, `Editor`, and `Viewer`.
- Assume collaborators must already have accounts for MVP sharing flows.
- Keep reservations and itinerary items as separate modules and separate persistence concerns.
- Route reminder scheduling through the API and reminder execution through Service Bus plus the worker.
- Keep PostgreSQL as the only system of record for application state.
- Keep Clerk responsible for identity and the API responsible for authorization.
- Keep email as the required reminder channel; treat in-app notifications as optional follow-on work.
- Add detail in downstream docs, but do not reopen these component boundaries without an explicit architecture change.
