# Async Reminders and Notifications for Travel Planner MVP

## Purpose

This document defines the asynchronous reminders and notifications subsystem for the Travel Planner MVP. It builds on:

- `docs/architecture/01-system-overview.md`
- `docs/architecture/02-domain-model.md`
- `docs/architecture/03-api-design.md`
- `docs/architecture/05-auth-security.md`

The goal is to keep reminder configuration simple for users while making delivery operationally safe, retry-aware, and consistent with the modular-monolith architecture:

- the API owns synchronous reminder creation, update, cancellation, and persistence
- Azure Service Bus owns due-message delivery timing and retry transport behavior
- the worker owns delivery execution and delivery outcome recording
- PostgreSQL remains the system of record for reminder and notification state

## Design Principles

- Keep reminder configuration trip-scoped and authorization-aware.
- Keep delivery asynchronous; never send reminder email inline with an API request.
- Keep the system idempotent so retries or duplicate message delivery do not create duplicate user-visible notifications.
- Keep the MVP narrow: one reminder row per recipient and one active delivery channel, email.
- Keep the transport contract explicit and boring; do not introduce event-stream abstractions or multi-subscriber fan-out without a later need.
- Keep operational state in PostgreSQL, not only in queue metadata, so failures can be diagnosed and recovered safely.

## What kinds of reminders exist in MVP

The MVP supports three reminder kinds, derived from the reminder target the user selects:

1. **Itinerary item reminders**
   - Example: remind a traveler two hours before a museum booking or train departure.
   - Backed by `Reminder.itineraryItemId`.

2. **Reservation reminders**
   - Example: remind a traveler the evening before hotel check-in or a few hours before a flight.
   - Backed by `Reminder.reservationId`.

3. **Checklist item reminders**
   - Example: remind a traveler to pack medication or bring passports before leaving.
   - Backed by `Reminder.checklistItemId`.

MVP rules:

- A reminder targets exactly one of `itineraryItemId`, `reservationId`, or `checklistItemId`.
- A reminder has exactly one recipient user.
- Email is the only active delivery channel.
- Recurring reminders, digest reminders, and multi-recipient fan-out in one request are intentionally out of scope.

## Which entities can create reminders

At the domain level, reminders can be created for these trip-scoped entities:

- `ItineraryItem`
- `Reservation`
- `ChecklistItem`

At the application boundary, reminder creation is initiated only through the API endpoint:

- `POST /v1/trips/:tripId/reminders`

Authorization rules remain aligned with the existing API and security design:

- `owner` and `editor` trip members can create, update, and cancel reminders
- `viewer` trip members can read reminder and notification records but cannot mutate them
- the recipient must be an active `TripMember` of the same trip
- all referenced target records must belong to the same `tripId`

Non-creators by design:

- `Traveler` records do not create reminders because travelers are not authorization principals
- the worker does not create new reminder intent; it only executes previously scheduled reminders
- the SPA may initiate reminder requests, but the API is the only trusted component that actually creates reminder rows

## Scheduling model

### Canonical scheduling fields

The canonical due time is `Reminder.scheduledFor` in UTC.

`offsetMinutes` is optional metadata used to explain how `scheduledFor` was derived:

- for itinerary and reservation reminders, `offsetMinutes` may be relative to the target's time anchor
- for checklist reminders, `scheduledFor` will usually be chosen directly because checklist items do not inherently have a timestamp

The worker dispatches based on `scheduledFor`, not by recalculating the due time from the related entity during execution.

### Scheduling behavior

Creation flow:

1. API validates the reminder request and target ownership.
2. API computes or accepts `scheduledFor`.
3. API inserts the `Reminder` row with `status = scheduled`.
4. API enqueues a Service Bus message scheduled for `scheduledFor`.

Update flow:

1. API allows changes only while the reminder is still mutable.
2. API updates the reminder row and increments a scheduling revision.
3. API schedules a new queue message for the latest revision.
4. Older queued messages are allowed to arrive later but are treated as stale and ignored by the worker.

Cancel flow:

1. API marks the reminder as `cancelled` and sets `cancelledAt`.
2. The API does not rely on deleting an already scheduled Service Bus message as the primary control path.
3. If an old queued message still arrives, the worker rehydrates the reminder, sees `status = cancelled`, and exits without delivery.

### Why stale-message tolerance is preferred

Service Bus supports scheduled delivery, but building the MVP around in-place mutation or cancellation of already-scheduled transport messages adds unnecessary coupling to broker-specific handles. A simpler and safer MVP design is:

- database row is the source of truth
- queue message is a delivery trigger, not the canonical reminder state
- worker must re-check reminder status, due time, and scheduling revision before sending

This makes updates and cancellations safe even if previously scheduled messages still exist in the queue.

## Service Bus usage model

### Queue/topic choice

Use a **single queue** for due reminder dispatch work, not a topic.

Recommended queue:

- `reminder-dispatch`

Why queue over topic for MVP:

- there is one execution consumer role: the reminder worker
- delivery work does not need independent subscriber fan-out yet
- queue semantics are simpler to operate and reason about
- retries and dead-lettering remain straightforward

A topic becomes interesting only if later versions need multiple independent subscribers for the same due reminder event, such as email delivery, in-app notification creation, analytics fan-out, or third-party webhook emission. That complexity is intentionally deferred.

### Message timing model

The API publishes a **scheduled queue message** whose visibility time equals the reminder's `scheduledFor`.

This keeps the worker simple:

- it consumes work only when due
- it does not need a polling scheduler loop over the reminders table
- PostgreSQL remains authoritative for state, while Service Bus handles timing and transport retry

### Message shape

The queue message should stay small and reference durable state instead of copying user-facing content.

Recommended message body:

```json
{
  "messageType": "reminder.dispatch",
  "reminderId": "uuid",
  "tripId": "uuid",
  "recipientUserId": "uuid",
  "channel": "email",
  "scheduledFor": "2026-04-20T14:00:00Z",
  "scheduleVersion": 3,
  "createdAt": "2026-04-18T09:15:00Z",
  "correlationId": "uuid",
  "idempotencyKey": "reminder:uuid:version:3"
}
```

Recommended transport/application properties:

- `contentType = application/json`
- `subject = reminder.dispatch`
- `messageId = reminder:{reminderId}:v:{scheduleVersion}`
- `correlationId = correlationId`

Design notes:

- `reminderId` is the primary lookup key
- `scheduleVersion` lets the worker reject stale scheduled messages after updates
- `messageId` and `idempotencyKey` help with broker dedupe and traceability, but the database check remains authoritative
- trip and recipient IDs are included for logging, validation, and quick diagnostics without extra parsing

### Retry strategy

Use two layers of retry behavior:

1. **Queue transport retry**
   - Let Service Bus redeliver the same message when the worker fails the handler or abandons the message.
   - Use a bounded max delivery count configured on the queue.

2. **Application retry safety**
   - The worker rehydrates state from PostgreSQL on every attempt.
   - The worker must safely re-run without creating duplicate `Notification` records or duplicate email sends.
   - Transient downstream failures from email or database connectivity should result in message retry, not silent failure.

Recommended MVP posture:

- keep retries finite and platform-managed
- rely on Service Bus delivery count rather than inventing a custom retry queue chain
- persist the latest failure reason on `Reminder.lastError` and on the active `Notification` attempt record

### Poison/dead-letter handling assumptions

When max delivery count is exhausted, the message should land in the queue's dead-letter subqueue automatically.

Assumptions for MVP:

- dead-lettering is an operational condition, not a silent drop
- the reminder row remains in PostgreSQL for investigation
- the latest corresponding notification record should end in `dead_lettered` or `failed`, depending on how the final worker attempt is recorded before the broker moves the message
- operators investigate dead-lettered reminders through logs, traces, queue metrics, and database state
- end users do not get a self-service retry UI in MVP

The worker should emit structured telemetry on every final failed attempt so dead-letter analysis can tie together:

- reminder ID
- trip ID
- recipient user ID
- delivery count
- queue message ID
- correlation ID
- downstream provider outcome

## Worker responsibilities in Azure Container Apps

The worker runs in `apps/worker` on Azure Container Apps and is the only runtime that consumes the `reminder-dispatch` queue.

Core responsibilities:

1. Receive a due reminder message from Service Bus.
2. Parse and validate the message contract.
3. Load the `Reminder`, related target entity, trip context, recipient user, and any existing `Notification` state from PostgreSQL.
4. Verify the reminder is still actionable:
   - status is `scheduled`
   - recipient is still a valid trip member
   - message `scheduleVersion` matches the current reminder version
   - reminder has not already been dispatched
5. Transition the reminder into `processing` inside a guarded transaction or equivalent atomic check.
6. Create or resume a `Notification` delivery attempt record.
7. Send the email through Azure Communication Services Email.
8. Persist the outcome:
   - success -> mark reminder `dispatched`, notification `sent`
   - retryable failure -> update error fields and let the queue retry
   - terminal or dead-letter path -> mark relevant records accordingly
9. Emit traces, logs, and metrics for the full attempt.

The worker should not:

- create new reminder intent on its own
- bypass database state checks and trust queue messages blindly
- mutate unrelated trip content
- perform authorization decisions for user-initiated writes

### Container Apps scaling assumptions

Azure Container Apps may scale the worker horizontally. Because multiple worker instances may process redeliveries or race with stale scheduled messages, the design must assume concurrent consumers.

Required safety posture:

- all delivery-side effects must be idempotent
- database transitions must guard against duplicate dispatch
- worker logic must tolerate the same reminder being attempted more than once

## Delivery channels

### Email now

Email is the only required and active delivery channel in MVP.

Email delivery path:

- API stores `Reminder.channel = email`
- worker renders the reminder email payload from current trip and target data
- worker sends through Azure Communication Services Email
- worker records the delivery outcome in `Notification`

Why email only now:

- it matches the system overview and domain model
- it keeps the first async system small enough to operate
- it avoids multiplying state transitions and delivery-provider integrations

### In-app notification later if deferred

An in-app notification center is explicitly deferred.

If it is added later, it should be modeled as an additional delivery channel or a closely related system-managed artifact, not as a replacement for the current reminder record. Likely future changes:

- add `in_app` to allowed channels or create a separate notification presentation layer
- potentially switch from one queue to either multiple queues or a topic if one due reminder needs multiple independent delivery pipelines
- add user-facing read/unread state and retention policies

The MVP should not prebuild those abstractions now.

## Idempotency and duplicate suppression

Duplicate prevention is required in both the API and the worker.

### API-side idempotency

The API design already requires `POST /v1/trips/:tripId/reminders` to support `Idempotency-Key`.

API rules:

- repeated client retries with the same effective request and idempotency key must return the existing reminder outcome instead of creating a second reminder
- queue publish should happen in the same logical operation as reminder creation; if publish fails, the API returns `503 QUEUE_PUBLISH_FAILED`
- reminder updates and cancellation must be safe to repeat

### Worker-side duplicate suppression

The worker must assume at-least-once delivery from Service Bus.

Worker suppression rules:

- ignore the message if the reminder is already `dispatched`
- ignore the message if the reminder is `cancelled`
- ignore the message if the queue message `scheduleVersion` is older than the database version
- use a unique persistence constraint or guarded state transition so only one active dispatch wins
- ensure a duplicate worker attempt cannot create two successful `Notification` send records for the same reminder/version/channel combination

Recommended persistence strategy:

- add `scheduleVersion` to `Reminder`
- add `dispatchedVersion` or equivalent delivery key to `Notification`
- enforce a uniqueness rule for successful delivery per reminder + schedule version + channel

This gives the worker a concrete basis for rejecting stale or duplicate work.

## Failure handling and observability

### Failure handling

Expected failure classes:

- invalid or stale queue message
- reminder no longer actionable because it was updated or cancelled
- missing related data due to deletion or inconsistent state
- database connectivity or transaction failure
- email provider transient failure
- email provider terminal rejection

Handling posture:

- stale or cancelled messages should be treated as safe no-ops, logged at info level with structured context
- transient infrastructure failures should trigger queue retry
- terminal failures should leave an explicit failure trail on the reminder and notification records
- dead-lettered messages require operator investigation, not silent abandonment

### Observability

Instrument the subsystem with OpenTelemetry and send telemetry to Azure Monitor / Application Insights.

Minimum telemetry requirements:

- trace from SPA request -> API reminder mutation -> Service Bus publish -> worker consume -> email provider call
- structured logs for reminder creation, update, cancel, dispatch start, dispatch success, dispatch failure, and dead-letter outcomes
- metrics for:
  - reminders scheduled
  - reminders dispatched successfully
  - retry count
  - dead-letter count
  - delivery latency from `scheduledFor` to actual send attempt
  - provider failure rate

Required correlation fields:

- request ID
- correlation ID
- reminder ID
- trip ID
- recipient user ID
- notification ID
- queue message ID

Privacy and security rules remain unchanged:

- do not log raw bearer tokens or secrets
- avoid logging freeform note bodies or unnecessary personal content
- prefer IDs, statuses, and provider codes over full payload dumps

## Data model implications

The conceptual model in `02-domain-model.md` is directionally correct, but MVP implementation should add a few explicit fields to support safe async execution.

### Reminder additions or clarifications

Recommended additions:

- `scheduleVersion` - increments whenever the reminder schedule or actionable recipient changes
- `processingStartedAt?` - useful for diagnosing stuck processing
- `dispatchedAt?` - explicit reminder-level lifecycle timestamp
- `lastAttemptedAt?` - optional operational timestamp for troubleshooting

Clarifications:

- `status = scheduled | processing | dispatched | failed | cancelled` remains valid
- `scheduledFor` is the only dispatch-time source of truth
- reminder target foreign keys stay relational; do not replace them with freeform polymorphic strings

### Notification additions or clarifications

Recommended additions:

- `scheduleVersion` - the reminder version this notification attempt belongs to
- `attemptNumber` - monotonic counter per reminder version
- `providerStatusCode?` - normalized provider outcome code when available
- `deadLetteredAt?` - explicit timestamp when a message is known to have exhausted retries

Clarifications:

- `Notification` remains the delivery audit record
- one reminder may have multiple notification attempts over time
- end-user APIs should read notification history; they should not directly mutate it

### Indexing and uniqueness

Recommended persistence constraints:

- index on `Reminder(tripId, status, scheduledFor)`
- index on `Reminder(recipientUserId, scheduledFor)`
- unique or guarded constraint that prevents more than one successful notification for the same `reminderId + scheduleVersion + channel`

These additions are worth doing because queue delivery is at-least-once and Container Apps may scale workers horizontally.

## What remains synchronous in the API and what is queued

### Synchronous in the API

- validate the caller's auth and trip membership
- validate the reminder target and same-trip invariants
- create reminder rows
- update reminder rows while mutable
- cancel reminder rows
- list reminders and notifications
- publish scheduled dispatch work to Service Bus
- return immediate success or explicit failure to the client

### Queued

- due reminder consumption
- reminder execution and duplicate suppression checks
- email rendering and send
- retry handling after transient delivery failures
- dead-letter routing after retry exhaustion
- operational alerting and investigation workflows

### Boundary rule

The API is responsible for durable user intent and enqueueing work. The worker is responsible for all externally visible reminder delivery side effects.

## One end-to-end reminder flow

Example: a trip editor creates a hotel check-in reminder for another trip member.

1. The editor calls `POST /v1/trips/:tripId/reminders` with `recipientUserId`, `channel = email`, `scheduledFor`, and `reservationId`.
2. The API validates that:
   - the caller is an owner or editor
   - the recipient is an active `TripMember`
   - the reservation belongs to the same trip
3. The API inserts `Reminder { status = scheduled, reservationId, scheduledFor, scheduleVersion = 1 }`.
4. The API schedules a Service Bus queue message on `reminder-dispatch` with:
   - `reminderId`
   - `tripId`
   - `recipientUserId`
   - `scheduleVersion = 1`
   - `messageId = reminder:{id}:v:1`
5. The API returns the created reminder to the SPA immediately.
6. When `scheduledFor` arrives, the worker receives the message.
7. The worker loads the reminder and verifies it is still `scheduled`, not cancelled, and still on `scheduleVersion = 1`.
8. The worker creates or resumes a `Notification` attempt row and marks the reminder `processing`.
9. The worker sends the email through Azure Communication Services Email.
10. On success, the worker marks:
    - reminder `dispatched`
    - notification `sent`
    - timestamps such as `dispatchedAt`
11. On transient failure, the worker records the error and allows Service Bus to retry.
12. If retries are exhausted, the message lands in the dead-letter subqueue and the subsystem emits telemetry for operator follow-up.

## MVP simplifications

- Only three reminder target types exist: itinerary item, reservation, checklist item.
- Only one recipient is allowed per reminder row.
- Only email is an active delivery channel.
- The system uses one queue, not a topic-based fan-out topology.
- The queue message carries identifiers and revision data, not a full denormalized reminder payload.
- The worker always rehydrates current state from PostgreSQL before sending.
- Reminder updates and cancellation are implemented by state change plus stale-message rejection, not by complex broker-side message mutation.
- There is no end-user retry, resend, or dead-letter management UI.
- There are no recurring schedules, snooze flows, quiet hours, or channel preferences in MVP.

## Final design guidance

- Keep PostgreSQL as the source of truth for reminder and notification state.
- Keep Service Bus responsible for scheduled transport and retry delivery semantics.
- Keep the worker responsible for idempotent side effects and delivery recording.
- Keep reminder creation simple for the user, but make duplicate suppression and dead-letter handling explicit for operators.
- Keep the MVP focused on reliable email reminders before adding broader notification abstractions.
