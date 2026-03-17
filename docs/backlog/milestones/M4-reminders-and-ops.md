# M4 - Reminders and Ops

## Objective

Make the MVP operationally complete by adding asynchronous reminders and the deployment, observability, and release-hardening work required to run the product reliably in Azure.

## In-scope capabilities

- reminder CRUD for trip-scoped targets:
  - itinerary items
  - reservations
  - checklist items
- notification history read APIs tied to reminder delivery outcomes
- API-side reminder validation, idempotency, and Service Bus scheduled publish
- worker-side reminder execution with:
  - state rehydration from PostgreSQL
  - schedule-version checks
  - duplicate suppression
  - retry-safe processing
  - dead-letter-aware failure handling
- Azure Communication Services Email integration for reminder delivery
- Bicep completion for reminder and ops infrastructure:
  - Service Bus queue
  - Key Vault references
  - communications resources
  - worker scaling configuration
- release and operational readiness work:
  - dashboards
  - alerts
  - staging and production rollout flow
  - smoke checks for API, worker, and migrations
  - backup and restore validation expectations for PostgreSQL

## Implementation slices

1. extend persistence for reminder and notification lifecycle fields
2. add reminder API endpoints and idempotent queue publish behavior
3. implement queue contract and worker consumption flow
4. integrate email provider and persist delivery outcomes
5. wire metrics, traces, alerts, and release checks
6. validate staging-style end-to-end behavior before production promotion

## Out-of-scope items

- recurring reminders
- multi-recipient fan-out in a single reminder request
- in-app notification center
- end-user retry or dead-letter management UI
- SMS, push, or additional delivery channels
- quiet hours, snooze, or notification preference systems
- private networking, multi-region failover, or Kubernetes-based platform changes

## Dependencies

- completed `M1-foundation`
- completed `M2-trip-planning-core`
- completed `M3-collaboration-and-sharing`
- reminder design and async boundaries from `docs/architecture/06-async-reminders.md`
- Azure deployment and release-order rules from `docs/architecture/07-azure-deployment.md`
- observability and release gate expectations from `docs/architecture/08-observability-testing.md`

Execution note:

- `B009` must already provide the shared-dev worker deployment scaffold plus the baseline Azure observability resources before `B021`-`B024` begin.

## Completion criteria

- owners and editors can create, update, cancel, and list reminders for itinerary items, reservations, and checklist items
- viewers can read reminder and notification history but cannot mutate reminder state
- reminder creation persists the database row and schedules Service Bus work in the same logical operation, returning explicit failure when queue publish fails
- the worker can consume due messages, rehydrate current state, reject stale or cancelled reminders, and record successful or failed delivery attempts without duplicate sends
- the system records notification outcomes and exposes enough history for user and operator troubleshooting
- Application Insights dashboards and alerts exist for API health, worker health, reminder delivery, queue backlog, and dependency failures
- CI/CD supports dev auto-deploy plus approval-gated staging and production promotion with immutable artifacts
- rollout order for infra, database, API, worker, and frontend is validated and documented in the delivery workflow

## Risk notes

- Async reminders are the highest operational-risk part of the MVP. Duplicate suppression, idempotency, and dead-letter visibility are mandatory, not optional polish.
- Queue timing and deployment ordering can create subtle failures if migrations, API changes, and worker changes are rolled out out of order. Follow the documented infra -> migration -> API -> worker -> frontend sequence.
- It is easy to underinvest in telemetry. If traces, logs, and alerts are not implemented alongside reminder delivery, production failures will be slow to diagnose.
- The reminder stream is tightly ordered: ship schema and API intent first (`B021`), then queue scheduling and worker orchestration (`B022`), then provider delivery and notification-history fidelity (`B023`), then validate dashboards/alerts (`B024`), then lock release workflow and runbooks (`B025`).
