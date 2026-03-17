# Domain Model for Travel Planner MVP

## Purpose

This document defines the core domain model for the Travel Planner MVP. It turns the PRD and earlier architecture decisions into a concrete conceptual model that downstream API, Prisma, frontend, and backlog work should follow.

The model favors:

- explicit trip scoping
- strong relational integrity
- simple authorization rules
- minimal workflow state needed for reminders and sharing

## Domain Modeling Principles

- Every user-facing business record is scoped either to a `User` or a `Trip`.
- Authorization is never derived from travelers, itinerary items, or reservations; it comes from `TripMember`.
- Travelers are not user accounts.
- Reservations and itinerary items are related concepts but remain separate aggregates.
- Reminder delivery is asynchronous and audit-aware, so reminder and notification records are retained longer than ordinary content records.
- The MVP uses a normalized relational model over document-style JSON blobs whenever the data participates in business rules or authorization.

## Core Entities with Fields and Purpose

### User

Purpose:

- Represents the application-level user record linked to Clerk identity.
- Owns trips, checklist templates, and reminder recipients.

Core fields:

- `id` - internal UUID primary key
- `clerkUserId` - unique external identity key from Clerk
- `email` - unique login/contact email
- `displayName` - user-facing name
- `avatarUrl?` - optional profile image URL
- `defaultTimezone?` - optional personal default timezone
- `createdAt`
- `updatedAt`

### Trip

Purpose:

- The top-level planning aggregate for a specific trip.
- Owns most trip-scoped content and summary views.

Core fields:

- `id`
- `ownerUserId` - the canonical trip owner
- `name`
- `destinationSummary`
- `description?`
- `startDate`
- `endDate`
- `status` - `draft | upcoming | active | completed | archived`
- `timezone` - required trip timezone for itinerary ordering and reminder scheduling
- `currencyCode` - required single trip currency for MVP budget totals
- `archivedAt?`
- `deletedAt?`
- `createdAt`
- `updatedAt`

### TripMember

Purpose:

- Defines who can access a trip and at what permission level.
- Carries the simplified MVP sharing lifecycle.

Core fields:

- `id`
- `tripId`
- `userId`
- `role` - `owner | editor | viewer`
- `invitationStatus` - `accepted | revoked`
- `invitedByUserId?`
- `invitedAt?`
- `acceptedAt?`
- `revokedAt?`
- `createdAt`
- `updatedAt`

Design note:

- There is no separate invitation entity in the MVP.
- Sharing creates or updates `TripMember` directly.

### Traveler

Purpose:

- Represents a person traveling on the trip, whether or not they have an account.
- Supports family and dependent planning scenarios.

Core fields:

- `id`
- `tripId`
- `displayName`
- `relationshipLabel?` - optional label such as spouse, child, grandparent
- `isMinor` - boolean flag for planning context
- `contactEmail?`
- `contactPhone?`
- `notes?`
- `createdAt`
- `updatedAt`

### ItineraryItem

Purpose:

- Represents a planned item in the day-by-day trip agenda.

Core fields:

- `id`
- `tripId`
- `title`
- `date`
- `startTime?`
- `endTime?`
- `category` - `transport | lodging | activity | meal | reminder | custom`
- `location?`
- `notes?`
- `linkUrl?`
- `sortOrder`
- `createdByUserId`
- `createdAt`
- `updatedAt`

### Reservation

Purpose:

- Stores booking and confirmation details for transport, lodging, activities, and similar reservations.

Core fields:

- `id`
- `tripId`
- `title`
- `type` - `flight | hotel | transport | activity | restaurant | car_rental | other`
- `provider?`
- `confirmationCode?`
- `startAt`
- `endAt?`
- `locationText?`
- `notes?`
- `bookingUrl?`
- `itineraryItemId?` - optional explicit link to a related itinerary item
- `createdByUserId`
- `createdAt`
- `updatedAt`

Design note:

- Traveler associations are many-to-many and should be normalized with a supporting join table such as `ReservationTraveler`.

### Checklist

Purpose:

- Represents a trip-scoped checklist such as packing, pre-departure, or documents.

Core fields:

- `id`
- `tripId`
- `name`
- `description?`
- `kind` - user-facing checklist grouping label
- `sourceTemplateId?`
- `createdByUserId`
- `createdAt`
- `updatedAt`

### ChecklistItem

Purpose:

- Represents a single actionable item inside a checklist.

Core fields:

- `id`
- `checklistId`
- `title`
- `notes?`
- `sortOrder`
- `isCompleted`
- `completedAt?`
- `completedByUserId?`
- `createdAt`
- `updatedAt`

### ChecklistTemplate

Purpose:

- Stores reusable checklist definitions owned by a user and copied into trips.

Core fields:

- `id`
- `ownerUserId`
- `name`
- `description?`
- `kind`
- `createdAt`
- `updatedAt`

Design note:

- Template item definitions should be stored as child rows inside the template aggregate, for example through a supporting `ChecklistTemplateItem` table, even though that child entity is not part of the core entity list for this document.

### Note

Purpose:

- Stores freeform trip information that does not fit structured entities.

Core fields:

- `id`
- `tripId`
- `title?`
- `body`
- `createdByUserId`
- `createdAt`
- `updatedAt`

### Link

Purpose:

- Stores important external URLs associated with a trip.

Core fields:

- `id`
- `tripId`
- `title`
- `url`
- `description?`
- `createdByUserId`
- `createdAt`
- `updatedAt`

### BudgetEntry

Purpose:

- Stores estimated and actual spending items for a trip.

Core fields:

- `id`
- `tripId`
- `title`
- `category` - `transport | lodging | food | activities | shopping | misc`
- `estimatedAmount?`
- `actualAmount?`
- `notes?`
- `reservationId?`
- `itineraryItemId?`
- `createdByUserId`
- `createdAt`
- `updatedAt`

Design note:

- Currency is inherited from `Trip.currencyCode` in the MVP.

### Reminder

Purpose:

- Represents a scheduled notification request for one trip-scoped target and one recipient user.

Core fields:

- `id`
- `tripId`
- `recipientUserId`
- `createdByUserId`
- `channel` - `email`
- `scheduledFor`
- `offsetMinutes?` - optional relative offset used to derive `scheduledFor`
- `status` - `scheduled | processing | dispatched | failed | cancelled`
- `itineraryItemId?`
- `reservationId?`
- `checklistItemId?`
- `lastError?`
- `cancelledAt?`
- `createdAt`
- `updatedAt`

Design note:

- Exactly one of `itineraryItemId`, `reservationId`, or `checklistItemId` must be present.
- Group notifications are modeled as multiple reminder rows, not one reminder with multiple recipients.

### Notification

Purpose:

- Captures the delivery record for a reminder sent through an external channel.

Core fields:

- `id`
- `tripId`
- `reminderId`
- `recipientUserId`
- `channel` - `email`
- `recipientAddress`
- `deliveryStatus` - `pending | sent | failed | dead_lettered | cancelled`
- `providerMessageId?`
- `dispatchedAt?`
- `deliveredAt?`
- `failedAt?`
- `failureReason?`
- `createdAt`
- `updatedAt`

## Entity Relationships

### Primary relationships

- One `User` owns many `Trip` records through `Trip.ownerUserId`.
- One `Trip` has many `TripMember` rows.
- One `User` can be a member of many trips through `TripMember`.
- One `Trip` has many `Traveler` records.
- One `Trip` has many `ItineraryItem` records.
- One `Trip` has many `Reservation` records.
- One `Trip` has many `Checklist` records.
- One `Checklist` has many `ChecklistItem` records.
- One `User` has many `ChecklistTemplate` records.
- One `Trip` has many `Note` records.
- One `Trip` has many `Link` records.
- One `Trip` has many `BudgetEntry` records.
- One `Trip` has many `Reminder` records.
- One `Reminder` has many `Notification` records over time if retries or re-dispatches need to be recorded separately.

### Supporting normalized relationships

- `Reservation` to `Traveler` is many-to-many through a supporting join table such as `ReservationTraveler`.
- `ChecklistTemplate` to template items is one-to-many through a supporting child table such as `ChecklistTemplateItem`.
- `BudgetEntry` optionally references either one `Reservation` or one `ItineraryItem`.
- `Reminder` references exactly one of `Reservation`, `ItineraryItem`, or `ChecklistItem`.

## Normalized Conceptual Schema

```text
User
  1 -> * Trip (ownerUserId)
  1 -> * ChecklistTemplate
  1 -> * TripMember

Trip
  1 -> * TripMember
  1 -> * Traveler
  1 -> * ItineraryItem
  1 -> * Reservation
  1 -> * Checklist
  1 -> * Note
  1 -> * Link
  1 -> * BudgetEntry
  1 -> * Reminder

Checklist
  1 -> * ChecklistItem

ChecklistTemplate
  1 -> * ChecklistTemplateItem (supporting child)

Reservation
  * -> * Traveler via ReservationTraveler

BudgetEntry
  0..1 -> Reservation
  0..1 -> ItineraryItem

Reminder
  * -> 1 User (recipientUserId)
  0..1 -> Reservation
  0..1 -> ItineraryItem
  0..1 -> ChecklistItem
  1 -> * Notification
```

## Ownership and Authorization Implications

- `User` is self-scoped, but domain access is mostly determined through trip membership.
- `Trip.ownerUserId` identifies the canonical owner responsible for deletion, sharing, and owner-only actions.
- `TripMember` is the only source of truth for trip access.
- `Traveler` does not grant access and must never be used as an authorization signal.
- `ChecklistTemplate` is owned by a user, not by a trip.
- All other core content entities inherit trip-level authorization from their `tripId`.
- `Reminder.recipientUserId` must always refer to an active trip member of the same trip.
- `Notification` is system-managed and should not be directly editable by end users.

Role implications:

- Owners can share trips, revoke access, archive trips, and delete trips.
- Editors can mutate trip-scoped planning content but cannot delete the trip or change ownership.
- Viewers can read trip-scoped content but cannot create, update, or delete it.

## Lifecycle States

### Trip Status

- `draft` - newly created, not yet ready for the active planning timeline
- `upcoming` - future trip with planning underway
- `active` - trip currently in progress
- `completed` - trip finished but still visible
- `archived` - hidden from primary active views but retained

Deletion is separate from status and represented by `deletedAt` for soft-deleted trips.

### Invitation Status

MVP rule:

- invitation state lives on `TripMember.invitationStatus`
- allowed values are `accepted` and `revoked`
- a `pending` state is intentionally omitted because sharing requires an existing account and membership is granted immediately

### Reminder Status

- `scheduled` - ready and waiting for due time
- `processing` - currently being handled by the worker
- `dispatched` - notification work was sent successfully
- `failed` - terminal failure after retry policy exhaustion
- `cancelled` - reminder intentionally cancelled before dispatch

### Notification Delivery Status

- `pending` - created but not yet sent
- `sent` - accepted by the delivery provider
- `failed` - send attempt failed and may still be retried
- `dead_lettered` - message exhausted retry policy and requires operator attention
- `cancelled` - cancelled before delivery

## Business Invariants

- Every `Trip` must have exactly one `ownerUserId`.
- Every `Trip` must have exactly one active `TripMember` with role `owner`, and that row must reference `Trip.ownerUserId`.
- A given `User` can have at most one `TripMember` row per trip.
- `Trip.endDate` must be on or after `Trip.startDate`.
- Viewers cannot mutate trip-scoped content.
- Only owners can delete or archive trips and manage trip sharing.
- `Traveler.tripId`, `ItineraryItem.tripId`, `Reservation.tripId`, `Checklist.tripId`, `Note.tripId`, `Link.tripId`, `BudgetEntry.tripId`, `Reminder.tripId`, and `Notification.tripId` must all match the trip context of their parent associations.
- A `ChecklistItem` cannot exist without a `Checklist`.
- A `Reservation` cannot exist without a `Trip`.
- A `BudgetEntry` may reference at most one of `reservationId` or `itineraryItemId`.
- A `Reminder` must reference exactly one of `reservationId`, `itineraryItemId`, or `checklistItemId`.
- A `Reminder.recipientUserId` must be an active `TripMember` of the same trip.
- A `Reminder` in `cancelled` state must not dispatch notifications.
- A `Notification` must reference the same trip and recipient as its parent `Reminder`.
- Checklist completion is derived from `ChecklistItem.isCompleted` state at the application level; do not add a redundant aggregate completion field unless a later design justifies it.
- Reservations do not automatically create itinerary items.

## Domain Events Worth Modeling

These events are worth modeling internally even if the first implementation emits them as application events rather than a full event-stream architecture:

- `trip.created`
- `trip.archived`
- `trip.deleted`
- `trip_member.invited`
- `trip_member.revoked`
- `traveler.added`
- `itinerary_item.created`
- `reservation.created`
- `checklist.created`
- `checklist.template_applied`
- `budget_entry.recorded`
- `reminder.scheduled`
- `reminder.cancelled`
- `reminder.dispatched`
- `notification.delivery_failed`

Event priorities:

- `trip.created`, `trip_member.invited`, `reminder.scheduled`, and `reminder.dispatched` are the highest-value events for MVP observability and async coordination.

## Recommended IDs and Timestamps Conventions

### IDs

- Use UUIDs for all core entity primary keys.
- Keep IDs immutable and opaque.
- Store external provider identifiers separately, for example `clerkUserId` and `providerMessageId`.
- Do not encode business meaning into IDs.

### Timestamps

- Every core entity gets `createdAt` and `updatedAt`.
- Store timestamps in UTC at the persistence boundary.
- Return API timestamps as ISO 8601 / RFC 3339 strings.
- Use explicit lifecycle timestamps such as `acceptedAt`, `revokedAt`, `completedAt`, `archivedAt`, `deletedAt`, `cancelledAt`, `dispatchedAt`, and `failedAt` where the event matters.
- Do not overload `updatedAt` to represent business lifecycle changes.

## Soft Delete vs Hard Delete Decisions

### Soft delete or status-based retention

- `Trip` - soft delete through `deletedAt`; collaborative data should not disappear immediately
- `TripMember` - revoke membership through `invitationStatus = revoked` and `revokedAt`
- `Reminder` - cancel rather than hard delete after scheduling has occurred
- `Notification` - retain for audit and operational troubleshooting
- `User` - user deletion is out of scope for MVP; retain the local user record even if upstream auth state changes

### Hard delete

- `Traveler` - hard delete unless blocked by active references that must be cleared first
- `ItineraryItem` - hard delete
- `Reservation` - hard delete
- `Checklist` - hard delete with cascading checklist items
- `ChecklistItem` - hard delete
- `ChecklistTemplate` - hard delete because trip checklists are copied from templates, not live-linked
- `Note` - hard delete
- `Link` - hard delete
- `BudgetEntry` - hard delete

Implementation note:

- When hard-deleting content referenced by reminders, cancel affected reminders first or enforce referential cleanup in a transaction.

## Fields That Should Be Optional vs Required

### Generally required

- all IDs and foreign keys that define ownership or scope
- user email and Clerk linkage
- trip name, destination summary, dates, timezone, currency, and status
- trip member role and invitation status
- itinerary item title, date, category, and sort order
- reservation title, type, and start time
- checklist name and checklist item title
- link title and URL
- budget entry title and category
- reminder recipient, channel, scheduled time, status, and exactly one target reference
- notification reminder linkage, recipient, channel, address, and delivery status

### Generally optional

- descriptive metadata such as notes, descriptions, URLs, provider names, and confirmation codes when the PRD marks them optional
- traveler contact details
- itinerary times and location details
- reservation end time, booking link, and linked itinerary item
- checklist source template linkage
- note title
- budget estimated or actual amount when the other value is the only one currently known
- reminder offset metadata and last error text
- provider delivery IDs and failure details on notifications

## MVP Simplifications

- Sharing only works with users who already have accounts.
- There is no separate invitation entity and no pending invitation workflow.
- `Traveler` remains separate from `User` and `TripMember`.
- Trips use a single currency code for budget totals.
- Reservations and itinerary items remain separate and only link explicitly when useful.
- Reminders are one-recipient, one-target records.
- Email is the only required reminder channel.
- Notifications are operational records, not a full in-app notification center.
- Checklist templates are copied into trips; edits to a template do not retroactively change existing trip checklists.

## Rules Agents Must Preserve During Implementation

- Do not merge `Traveler`, `User`, and `TripMember` into one concept.
- Do not bypass `TripMember` for authorization decisions.
- Do not introduce a pending invitation flow unless a later architecture decision explicitly adds it.
- Do not auto-create itinerary items from reservations by default.
- Do not model reminders as unscoped records; every reminder must belong to a trip and a recipient user.
- Do not use a polymorphic `targetType` plus freeform `targetId` string for reminders when relational foreign keys can preserve integrity.
- Do not store trip-scoped structured content as generic JSON blobs when relational tables are appropriate.
- Do not hard-delete reminders or notifications after they enter the async workflow.
- Do not allow viewer-role users to mutate trip content.
- Do not let budget entries reference both a reservation and an itinerary item at the same time.
- Do not let a trip exist without a matching owner membership.
