# Observability and Testing Strategy for Travel Planner MVP

## Purpose

This document defines the MVP quality strategy for Travel Planner across testing, observability, release gates, and task-level completion criteria. It builds on the full architecture set and consolidates the quality expectations that implementation tasks must follow.

The goal is to keep quality work:

- fast enough for daily local development
- targeted to the product's real risks
- explicit about what must be tested
- explicit about what must be observable in production
- practical for a small team shipping a modular monolith on Azure

## Quality principles

- Prefer many fast tests and a small number of critical end-to-end tests.
- Test business rules closest to the code that enforces them.
- Treat authorization, validation, and cross-trip integrity as first-class test concerns.
- Treat async reminder delivery and failure handling as first-class observability concerns.
- Treat logs, traces, and metrics as part of the feature, not post-release cleanup.
- Prefer small, explicit fixtures and deterministic clocks over large opaque datasets.
- Keep privacy rules intact in tests and telemetry; do not normalize over-logging just because a system is in non-prod.

## Test pyramid for this project

The project should follow a pragmatic pyramid:

1. **Unit tests**
   - highest volume
   - fastest feedback
   - run on nearly every code change

2. **Integration tests**
   - medium volume
   - focus on module boundaries, persistence, HTTP handlers, and async orchestration seams

3. **API contract tests**
   - smaller focused layer
   - validate request/response shapes, status codes, error envelopes, and auth/validation expectations

4. **Frontend component and page tests**
   - moderate volume
   - focus on meaningful UI behavior, screen states, and role-aware rendering

5. **End-to-end tests**
   - smallest layer
   - cover only the most important journeys and release-critical flows

### Tooling by layer

- frontend unit/component/page tests: `Vitest` + `React Testing Library`
- backend unit/integration/API tests: `Jest` + `Supertest`
- end-to-end tests: `Playwright`
- type safety gate: `tsc --noEmit`

## What each layer must cover

### Unit tests

Unit tests should cover isolated logic with minimal I/O:

- domain rules and guards
- date and scheduling calculations
- normalization helpers such as email or identifier handling
- permission evaluation helpers
- mapper and formatter utilities
- small React components with meaningful state logic
- form schema logic and client-side validation helpers

Unit tests should not be the only evidence for:

- route-level authorization
- Prisma persistence behavior
- OpenAPI or HTTP contract behavior
- reminder queue integration

### Integration tests

Integration tests should cover realistic behavior across module boundaries:

- NestJS services using Prisma against a test database
- controller-to-service flows for reads and mutations
- trip membership enforcement across resources
- cross-trip reference rejection
- mutation side effects such as reminder scheduling requests
- worker message handling against realistic persistence state

Integration tests are the primary place to validate:

- data invariants
- authorization enforcement
- retry-safe async coordination
- database-backed lifecycle transitions

### API contract tests

API contract tests should verify the public HTTP surface defined in `03-api-design.md` and the OpenAPI generation strategy.

They must cover:

- request body and query validation
- success payload shape
- error envelope shape
- auth outcomes such as `401`, `403`, and `404` in access-controlled cases
- idempotency behavior for endpoints that require `Idempotency-Key`
- backward-compatible response changes for existing endpoints

Recommended approach:

- generate OpenAPI from NestJS DTOs and decorators
- validate selected routes against the generated contract in CI
- keep DTO classes as the source of truth

### Frontend component and page tests

These tests should cover:

- loading, empty, success, and inline error states
- form validation and submit-state behavior
- route-level or page-level rendering with mocked API responses
- role-aware action visibility and disabled states
- mobile-sensitive rendering behavior where layout or interaction changes materially
- mutation success flows and query invalidation behavior

These tests should not try to prove backend authorization. They should only prove UI affordances and user-visible behavior.

### End-to-end tests

End-to-end tests should cover only critical journeys:

- sign in and session bootstrap
- create trip
- edit trip metadata
- add itinerary item
- create reservation
- create checklist and complete item
- create budget entry
- share trip with an existing user
- create and observe a reminder in a safe non-destructive environment if practical

E2E tests should not become the main regression safety net for routine CRUD branches that unit and integration tests can cover faster.

## Minimum required tests per task type

The following minimums apply unless a task is documentation-only or purely non-functional infrastructure work.

| Task type | Minimum required tests |
|---|---|
| Pure frontend presentational change | Type-check plus component/page test if behavior or state changes materially |
| Frontend form or screen flow change | Type-check plus component/page test covering validation, loading, success, and error states |
| Frontend role-aware or mobile-sensitive change | Type-check plus regression test for role gating or mobile-specific layout behavior |
| Backend read endpoint | Unit or integration coverage for query logic, plus API test for success and relevant auth/not-found behavior |
| Backend mutation endpoint | Integration/API tests for success, validation failure, authorization failure, and side effects |
| Auth or permission change | Tests for owner, editor, viewer, and non-member cases where relevant |
| Prisma schema or migration change | Migration validation plus integration tests that exercise the new schema behavior |
| Reminder or worker async change | Integration tests for scheduling/processing logic plus failure-path or retry-path coverage |
| API contract change | Contract test updates plus consumer-impact review |
| Bug fix | A regression test at the lowest reliable layer that would have caught the bug |
| Infrastructure/observability change | Validation of deployment config plus smoke verification or alert/dashboard update when behavior changes |

## Test data strategy

### Principles

- keep fixtures small and trip-scoped
- prefer builders/factories over giant JSON snapshots
- use synthetic data only
- make authorization state obvious in the test setup
- make time-dependent tests deterministic

### Recommended fixture model

Use reusable fixture builders for:

- users
- trips
- trip members by role
- travelers
- itinerary items
- reservations
- checklists and checklist items
- reminders and notifications

Fixture rules:

- every test should make `tripId` ownership obvious
- use explicit owner/editor/viewer/non-member combinations for authorization-sensitive scenarios
- do not hide important IDs or timestamps behind overly magical factory defaults
- freeze time when testing scheduling, deadlines, reminders, or retention logic

### Database test data

For backend integration tests:

- use a dedicated test database or isolated schema
- apply Prisma migrations before the suite or test phase
- reset data between suites or test cases using a predictable mechanism
- prefer seed helpers over hand-written SQL unless a specific migration test needs SQL

### Frontend test data

For frontend tests:

- mock Clerk session state
- mock API responses at the HTTP boundary
- keep fixtures aligned with the shared contracts and real endpoint shapes
- prefer narrowly scoped response fixtures over one global mega-fixture

### End-to-end data

For Playwright:

- use seeded shared-dev or dedicated test data only
- create ephemeral entities inside the test when setup is cheap
- avoid brittle assumptions about globally shared records
- use non-destructive reminder/email adapters in automated E2E runs unless the environment is explicitly safe

## Local development test workflow

The local workflow should optimize for fast feedback first, then broader confidence before merge.

### Inner loop

For the code a developer is actively changing:

- run targeted unit/component/integration tests
- run `tsc --noEmit`
- run linting for affected packages or apps

Expected local habits:

- frontend work should run the affected `Vitest` suite before handoff
- backend work should run the affected `Jest` or `Supertest` suite before handoff
- async reminder changes should run worker-related tests and at least one failure-path test

### Pre-push or pre-merge local check

Before opening or updating a PR, developers should run:

- lint
- type-check
- affected unit/component/integration tests
- any relevant contract tests

When the change affects a critical cross-app flow, also run:

- targeted Playwright smoke coverage locally or against shared dev

### Local environment expectations

- run `web`, `api`, and `worker` through Turborepo tasks
- run PostgreSQL locally through Docker Compose or equivalent
- use seeded local data for realistic trip scenarios
- avoid manual database edits; use Prisma migrations and seed scripts
- prefer local execution over cloud dependency, except for Azure-specific integration points such as Service Bus behavior

## OpenTelemetry instrumentation expectations

OpenTelemetry is required for backend and worker services. Frontend telemetry may be lighter, but request correlation must still be preserved end to end.

### Common resource attributes

Every service should emit at least:

- `service.name`
- `service.version`
- `deployment.environment`
- `service.instance.id` where available

### API instrumentation expectations

Instrument:

- incoming HTTP requests
- outgoing HTTP calls to external services where relevant
- Prisma database queries or database dependency spans
- Service Bus publish operations
- auth/session sync critical paths
- important mutations such as trip creation, sharing changes, and reminder mutations

### Worker instrumentation expectations

Instrument:

- Service Bus receive, processing, completion, abandon, and dead-letter flows
- PostgreSQL lookups and writes during reminder handling
- email provider calls
- retry attempts
- queue-to-send latency and processing duration

### Frontend instrumentation expectations

Keep frontend telemetry intentionally lighter than backend telemetry.

Recommended minimum:

- page or route load timing
- uncaught client errors
- request correlation headers to the API where supported

Do not treat frontend browser telemetry as the source of truth for business operations. API and worker telemetry remain authoritative.

## Logging standards

### Structured logging

All server-side logs should be structured and machine-queryable.

Each meaningful log entry should include where relevant:

- timestamp
- severity
- service name
- environment
- request ID or correlation ID
- authenticated user ID if known
- trip ID
- resource ID such as reminder ID or notification ID
- operation name
- outcome status

### Logging levels

- `debug` - local troubleshooting or narrowly scoped non-prod diagnostics
- `info` - normal lifecycle events and successful important operations
- `warn` - unexpected but recoverable conditions
- `error` - failed operations that need investigation

### Logging standards for mutations

Log at least one structured event for:

- mutation start or acceptance when operationally important
- mutation success
- mutation failure

High-value mutation examples:

- trip creation
- member add/revoke
- reminder create/update/cancel
- destructive deletes

### Logging standards for async processing

For reminder processing, log:

- message received
- stale or cancelled message no-op
- dispatch start
- dispatch success
- retryable failure
- terminal failure or dead-letter outcome

### Logging privacy rules

- never log secrets, tokens, raw auth headers, or connection strings
- avoid logging note bodies, reservation confirmation codes, or other unnecessary personal content
- prefer IDs, enums, counts, and provider codes over raw payload dumps
- keep high-cardinality freeform data out of normal log fields

## Metrics and traces expectations

### Core metrics

API metrics:

- request count
- error rate
- latency percentiles by route group
- auth failure rate
- dependency failure rate

Worker metrics:

- messages received
- messages completed
- retry count
- dead-letter count
- queue lag or scheduled-to-processed latency
- reminder dispatch success and failure counts
- email provider failure rate

Database and dependency metrics:

- PostgreSQL dependency latency and failure count
- Service Bus publish/receive failures
- email provider dependency latency and failure count

### Trace expectations

Critical traces should connect:

- browser request or route interaction
- API request handling
- database work
- Service Bus publish
- worker message handling
- downstream email provider dependency

High-value traceable flows:

- session sync
- trip creation
- trip sharing
- reminder scheduling
- reminder dispatch

### Trace correlation rules

Correlation should preserve:

- request ID
- correlation ID
- reminder ID
- trip ID
- recipient user ID where relevant
- queue message ID for async work

## Observability requirements for mutations, async processing, and failures

### Mutations

Every important mutation must have:

- at least one trace span covering the mutation path
- structured success/failure logging
- request/latency/error metrics at the API level

At minimum, this applies to:

- trip creation and deletion
- sharing changes
- reservation creation/update/delete
- checklist item completion
- budget mutations
- reminder create/update/cancel

### Async processing

Every reminder-processing path must emit:

- queue receive span
- reminder-processing span
- dependency spans for DB and email
- success or failure logs
- retry/dead-letter metrics

### Failures

All failures that affect user-visible behavior or operational reliability must be observable through:

- an error log with correlation context
- a trace with the failed dependency or domain step
- a metric or alertable signal when failure volume matters

Examples:

- repeated authorization failures
- queue publish failures
- reminder dead-letter events
- email provider outages
- migration failures during release

## Key dashboards and alerts to create in Application Insights

### Dashboards

1. **API health dashboard**
   - request volume
   - p50/p95 latency
   - 4xx/5xx rate
   - top failing routes

2. **Worker and reminders dashboard**
   - messages processed
   - retry count
   - dead-letter count
   - reminder dispatch success rate
   - scheduled-to-send latency

3. **Dependency health dashboard**
   - PostgreSQL latency/failure rate
   - Service Bus publish/receive latency/failure rate
   - email provider latency/failure rate

4. **Security and authorization dashboard**
   - auth failure rate
   - repeated forbidden outcomes
   - sharing-related mutation activity

5. **Release confidence dashboard**
   - error rate after deploy
   - request latency after deploy
   - reminder failure rate after deploy
   - recent deployment markers

### Alerts

Create alerts for at least:

- API 5xx error rate above threshold
- sustained p95 API latency above threshold
- worker dead-letter count above threshold
- reminder dispatch failure rate above threshold
- queue backlog age or depth above threshold
- PostgreSQL dependency failures above threshold
- email provider failure spike
- no worker processing activity when due reminders should exist

Thresholds can be tuned by environment, but the alert categories should exist from the start.

## Release quality gates

### Pull request gates

Every PR should pass:

- lint
- type-check
- affected unit/component/integration tests
- contract validation when API surface changes
- Bicep validation when infra changes

### Main-branch or shared-dev gates

Before treating `main` as releasable:

- build artifacts successfully for `web`, `api`, and `worker`
- migrations validate cleanly
- shared-dev deployment smoke checks pass
- no unresolved critical alerts caused by the change

### Staging gates

Before production promotion:

- staging deployment succeeds
- critical Playwright journeys pass
- schema migrations succeed
- API and worker smoke checks pass
- dashboards show no critical regressions

### Production gates

Before or during production rollout:

- manual approval is present
- rollback path is identified
- previous critical alerts are acknowledged or resolved
- no known contract or migration incompatibility remains

## Definition of done checklist for tasks

A task is not done unless the relevant items below are satisfied.

### All code changes

- type-check passes for affected code
- appropriate automated tests are added or updated
- user-visible failures are handled explicitly
- documentation is updated when architecture or workflow assumptions changed

### Backend tasks

- validation and authorization paths are covered
- structured logs, traces, and metrics exist for important mutations or failures
- API contract changes are reflected in DTOs and verified

### Frontend tasks

- loading, empty, success, and error states are handled
- mobile-sensitive behavior is considered
- role-aware UI behavior is tested when relevant

### Async or worker tasks

- idempotency or retry behavior is tested where relevant
- success, retry, and failure paths are observable
- dead-letter or terminal failure behavior is not silent

### Data or migration tasks

- Prisma schema and migrations are updated
- migration impact is tested or validated
- rollback or restore implications are considered for risky changes

### Infrastructure or deployment tasks

- Bicep changes are included when infrastructure requirements change
- deployment validation is updated
- dashboards or alerts are updated if operational behavior changed

## Final design guidance

- Keep the majority of coverage in fast unit, integration, and page-level tests.
- Keep API contract and authorization behavior explicit and regression-protected.
- Keep Playwright focused on the smallest set of critical user journeys.
- Keep OpenTelemetry, logs, traces, and metrics as required implementation work for important operations.
- Keep release gates simple but real: passing tests, healthy deploys, and visible operational signals.
