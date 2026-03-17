# API Design for Travel Planner MVP

## Purpose

This document defines the HTTP API for the Travel Planner MVP. It assumes:

- React SPA frontend
- NestJS REST API backend
- Clerk-managed authentication
- PostgreSQL as the system of record
- Azure Service Bus for asynchronous reminder delivery

The API is designed for the SPA only. It is versioned, JSON-based, resource-oriented, and shaped for straightforward NestJS controller and DTO implementation.

## API Base Conventions

- Base path: `/v1`
- Content type: `application/json`
- Auth: `Authorization: Bearer <Clerk JWT>` on all SPA-facing endpoints
- Time format: ISO 8601 / RFC 3339 strings
- Money format: decimal strings, not floating-point JSON numbers
- ID format: UUID strings

### Response Envelopes

Single resource response:

```json
{
  "data": {}
}
```

List response:

```json
{
  "data": [],
  "pageInfo": {
    "nextCursor": null,
    "hasMore": false
  }
}
```

Mutation responses return the updated resource inside `data`.

## Endpoint Naming Conventions

- Use plural nouns for collections: `/trips`, `/reservations`, `/budget-entries`
- Use nested resources for trip-scoped entities: `/trips/:tripId/travelers`
- Use `/me` for current-user profile endpoints
- Prefer canonical resource paths over verb-based names
- Use `PATCH` for partial updates
- Use `DELETE` for user-facing deletion even if the backend applies soft delete internally
- Allow explicit action subpaths only when they protect workflow rules better than a general `PATCH`, for example reminder cancellation

## DTO Design Guidance

- Define separate DTOs for create, update, list item, and detail responses
- Do not expose Prisma models directly
- Use string enums in DTOs for domain states and categories
- Keep request DTOs flat unless nesting reflects a true aggregate structure
- Represent optional fields explicitly as nullable or omitted consistently per DTO
- Return derived authorization context where useful, for example `currentUserRole` on trip detail responses
- Return supporting IDs as arrays where the domain model uses join tables internally, for example `travelerIds` on reservations
- Use dedicated list item DTOs for collections to keep payloads small on mobile

## Pagination, Filtering, and Sorting Strategy

### Pagination

- Use cursor pagination for list endpoints that can grow materially: trips, checklist templates, notifications
- Support `limit` with a default of `20` and a maximum of `100`
- Use `cursor` as an opaque token derived from the primary sort tuple
- Smaller trip-scoped collections may return a single page in MVP, but should still use the common list envelope

### Filtering

Use query parameters only. Examples:

- trips: `status`, `startsAfter`, `startsBefore`
- itinerary items: `date`, `startDate`, `endDate`, `category`
- reservations: `type`, `startsAfter`, `startsBefore`
- checklists: `kind`
- reminders: `status`, `targetType`
- notifications: `deliveryStatus`

### Sorting

- Default sort is stable and explicit per resource
- Trips: `startDate:asc`, then `id`
- Itinerary items: `date:asc`, `startTime:asc`, `sortOrder:asc`
- Reservations: `startAt:asc`, then `id`
- Notes and links: `updatedAt:desc`
- Budget entries: `createdAt:desc`
- Notifications: `createdAt:desc`

Expose `sort` only where multiple user-meaningful orders are needed.

## Validation Approach

- Use NestJS global `ValidationPipe` with `transform`, `whitelist`, and `forbidNonWhitelisted`
- Model request DTOs with `class-validator` and `class-transformer`
- Parse path IDs with `ParseUUIDPipe`
- Validate enums explicitly at DTO boundaries
- Enforce business invariants in services, not only in DTO validation
- Reject cross-trip references explicitly, for example a reminder cannot point at a checklist item from another trip
- Normalize strings such as email addresses before uniqueness checks

## API Error Envelope

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request body failed validation.",
    "status": 422,
    "requestId": "req_123",
    "details": [
      {
        "field": "startDate",
        "issue": "must be on or before endDate"
      }
    ]
  }
}
```

Error rules:

- `code` is stable and machine-readable
- `message` is human-readable
- `status` mirrors the HTTP status
- `requestId` is included for correlation
- `details` is optional and used mainly for validation failures

Common status usage:

- `401` - unauthenticated
- `403` - authenticated but not authorized
- `404` - resource not found in the caller's accessible scope
- `409` - conflict or invariant violation
- `422` - validation failure
- `503` - dependency unavailable, for example queue publish failure

## Idempotency Rules

- `POST /v1/session/sync` is naturally idempotent by authenticated Clerk identity
- `POST /v1/trips` is not idempotent by default; clients may retry only with an `Idempotency-Key` header
- `POST /v1/trips/:tripId/members` should be idempotent by `tripId + email`; repeated calls with the same effective membership return the current membership
- `POST /v1/trips/:tripId/reminders` should support `Idempotency-Key` to avoid duplicate reminder scheduling on client retry
- `DELETE` operations must be safe to repeat when the resource is already absent or already soft-deleted
- `POST /v1/trips/:tripId/reminders/:reminderId/cancel` must be safe to repeat once a reminder is already cancelled

## OpenAPI Generation Strategy in NestJS

- Use NestJS code-first OpenAPI generation with `@nestjs/swagger`
- Decorate DTOs with `ApiProperty` metadata and controllers with operation/response decorators
- Generate a single versioned spec from the running application in CI
- Publish `openapi.json` as a build artifact
- Keep DTO classes as the source of truth for request and response contracts
- Optionally generate frontend API types from the OpenAPI spec later, but do not make generated code the design source of truth

## What Should Be Synchronous vs Queued

### Synchronous

- all reads
- trip CRUD
- membership and sharing changes
- traveler CRUD
- itinerary CRUD
- reservation CRUD
- checklist and checklist item CRUD
- checklist template CRUD
- note and link CRUD
- budget entry CRUD
- reminder creation and update
- reminder cancellation

### Queued

- actual reminder dispatch
- email delivery
- retry handling for failed notification delivery
- dead-letter operational workflows

Rule:

- The API may synchronously create a reminder record and enqueue scheduling work, but actual delivery must never happen inline with the request.

## Resource Endpoints

### Auth / Session Integration Assumptions

Clerk owns sign-in, sign-out, password reset, and session issuance. The API does not expose login or logout endpoints.

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `POST` | `/v1/session/sync` | Ensure the local `User` record exists for the authenticated Clerk identity and return the current profile | Body: none | `SessionSyncResponse { user, membershipsSummary }` | Any authenticated user | `401 UNAUTHENTICATED`, `409 EMAIL_CONFLICT`, `503 DEPENDENCY_UNAVAILABLE` |
| `GET` | `/v1/me` | Get the current user's profile and summary context | Query: none | `UserProfileResponse { id, email, displayName, avatarUrl, defaultTimezone }` | Any authenticated user | `401 UNAUTHENTICATED`, `404 USER_NOT_FOUND` |
| `PATCH` | `/v1/me` | Update editable profile fields stored locally | `{ displayName?, defaultTimezone? }` | `UserProfileResponse` | Any authenticated user | `401 UNAUTHENTICATED`, `422 VALIDATION_FAILED` |

### Trips

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips` | List trips visible to the current user | Query: `{ limit?, cursor?, status?, startsAfter?, startsBefore?, sort? }` | `TripListResponse { data: TripSummaryDto[], pageInfo }` | Any authenticated user | `401 UNAUTHENTICATED`, `422 VALIDATION_FAILED` |
| `POST` | `/v1/trips` | Create a trip and owner membership | `{ name, destinationSummary, description?, startDate, endDate, timezone, currencyCode }` | `TripDetailResponse` | Any authenticated user | `401 UNAUTHENTICATED`, `422 VALIDATION_FAILED`, `409 INVALID_TRIP_DATES` |
| `GET` | `/v1/trips/:tripId` | Get a trip detail view | Path: `tripId` | `TripDetailResponse { trip, currentUserRole, counts }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND` |
| `PATCH` | `/v1/trips/:tripId` | Update trip metadata or status | `{ name?, destinationSummary?, description?, startDate?, endDate?, timezone?, currencyCode?, status? }` | `TripDetailResponse` | Owner or editor, except only owner may set `archived` | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED`, `409 INVALID_TRIP_STATE` |
| `DELETE` | `/v1/trips/:tripId` | Soft-delete a trip | Body: none | `DeleteResponse { id, deletedAt }` | Owner only | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `409 OWNER_ROLE_REQUIRED` |

### Trip Members / Sharing

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/members` | List trip members and roles | Query: none | `TripMemberListResponse { data: TripMemberDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND` |
| `POST` | `/v1/trips/:tripId/members` | Share a trip with an existing user account | `{ email, role }` | `TripMemberResponse` | Owner only | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `404 USER_NOT_FOUND`, `409 CANNOT_ADD_SECOND_OWNER`, `422 VALIDATION_FAILED` |
| `PATCH` | `/v1/trips/:tripId/members/:memberId` | Change a member's role | `{ role }` | `TripMemberResponse` | Owner only | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_OR_MEMBER_NOT_FOUND`, `409 CANNOT_CHANGE_LAST_OWNER` |
| `DELETE` | `/v1/trips/:tripId/members/:memberId` | Revoke a member's trip access | Body: none | `TripMemberResponse { id, invitationStatus: "revoked", revokedAt }` | Owner only | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_OR_MEMBER_NOT_FOUND`, `409 CANNOT_REMOVE_LAST_OWNER` |

### Travelers

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/travelers` | List travelers on a trip | Query: `{ limit?, cursor? }` | `TravelerListResponse { data: TravelerDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND` |
| `POST` | `/v1/trips/:tripId/travelers` | Add a traveler to a trip | `{ displayName, relationshipLabel?, isMinor?, contactEmail?, contactPhone?, notes? }` | `TravelerResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `PATCH` | `/v1/trips/:tripId/travelers/:travelerId` | Update traveler details | `{ displayName?, relationshipLabel?, isMinor?, contactEmail?, contactPhone?, notes? }` | `TravelerResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_OR_TRAVELER_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/travelers/:travelerId` | Remove a traveler | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_OR_TRAVELER_NOT_FOUND`, `409 TRAVELER_IN_USE` |

### Itinerary Items

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/itinerary-items` | List itinerary items for a trip or date range | Query: `{ date?, startDate?, endDate?, category?, sort? }` | `ItineraryItemListResponse { data: ItineraryItemDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 INVALID_DATE_RANGE` |
| `POST` | `/v1/trips/:tripId/itinerary-items` | Create an itinerary item | `{ title, date, startTime?, endTime?, category, location?, notes?, linkUrl?, sortOrder? }` | `ItineraryItemResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `GET` | `/v1/trips/:tripId/itinerary-items/:itemId` | Get one itinerary item | Path only | `ItineraryItemResponse` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 ITEM_NOT_FOUND` |
| `PATCH` | `/v1/trips/:tripId/itinerary-items/:itemId` | Update an itinerary item | `{ title?, date?, startTime?, endTime?, category?, location?, notes?, linkUrl?, sortOrder? }` | `ItineraryItemResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 ITEM_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/itinerary-items/:itemId` | Delete an itinerary item | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 ITEM_NOT_FOUND`, `409 ITEM_REFERENCED_BY_REMINDER` |

### Reservations

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/reservations` | List reservations | Query: `{ limit?, cursor?, type?, startsAfter?, startsBefore?, sort? }` | `ReservationListResponse { data: ReservationDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `POST` | `/v1/trips/:tripId/reservations` | Create a reservation | `{ title, type, provider?, confirmationCode?, startAt, endAt?, locationText?, notes?, bookingUrl?, itineraryItemId?, travelerIds? }` | `ReservationResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `404 RELATED_RESOURCE_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `GET` | `/v1/trips/:tripId/reservations/:reservationId` | Get one reservation | Path only | `ReservationResponse` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 RESERVATION_NOT_FOUND` |
| `PATCH` | `/v1/trips/:tripId/reservations/:reservationId` | Update a reservation | `{ title?, type?, provider?, confirmationCode?, startAt?, endAt?, locationText?, notes?, bookingUrl?, itineraryItemId?, travelerIds? }` | `ReservationResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 RESERVATION_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/reservations/:reservationId` | Delete a reservation | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 RESERVATION_NOT_FOUND`, `409 RESERVATION_REFERENCED_BY_REMINDER` |

### Checklists and Checklist Items

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/checklists` | List checklists for a trip | Query: `{ kind?, sort? }` | `ChecklistListResponse { data: ChecklistSummaryDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND` |
| `POST` | `/v1/trips/:tripId/checklists` | Create a checklist, optionally from a template | `{ name?, description?, kind, sourceTemplateId?, items? }` | `ChecklistDetailResponse { checklist, items }` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `404 TEMPLATE_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `GET` | `/v1/trips/:tripId/checklists/:checklistId` | Get one checklist with items | Path only | `ChecklistDetailResponse` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 CHECKLIST_NOT_FOUND` |
| `PATCH` | `/v1/trips/:tripId/checklists/:checklistId` | Update checklist metadata | `{ name?, description?, kind? }` | `ChecklistDetailResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 CHECKLIST_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/checklists/:checklistId` | Delete a checklist and its items | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 CHECKLIST_NOT_FOUND`, `409 CHECKLIST_REFERENCED_BY_REMINDER` |
| `POST` | `/v1/trips/:tripId/checklists/:checklistId/items` | Add a checklist item | `{ title, notes?, sortOrder? }` | `ChecklistItemResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 CHECKLIST_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `PATCH` | `/v1/trips/:tripId/checklists/:checklistId/items/:itemId` | Update or complete a checklist item | `{ title?, notes?, sortOrder?, isCompleted? }` | `ChecklistItemResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 ITEM_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/checklists/:checklistId/items/:itemId` | Delete a checklist item | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 ITEM_NOT_FOUND`, `409 ITEM_REFERENCED_BY_REMINDER` |

### Checklist Templates

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/checklist-templates` | List the current user's checklist templates | Query: `{ limit?, cursor?, kind?, sort? }` | `ChecklistTemplateListResponse { data: ChecklistTemplateSummaryDto[] }` | Any authenticated user | `401 UNAUTHENTICATED`, `422 VALIDATION_FAILED` |
| `POST` | `/v1/checklist-templates` | Create a reusable checklist template | `{ name, description?, kind, items: [{ title, notes?, sortOrder? }] }` | `ChecklistTemplateDetailResponse` | Any authenticated user | `401 UNAUTHENTICATED`, `422 VALIDATION_FAILED` |
| `GET` | `/v1/checklist-templates/:templateId` | Get a template with its items | Path only | `ChecklistTemplateDetailResponse` | Template owner only | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TEMPLATE_NOT_FOUND` |
| `PATCH` | `/v1/checklist-templates/:templateId` | Update template metadata or items | `{ name?, description?, kind?, items? }` | `ChecklistTemplateDetailResponse` | Template owner only | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TEMPLATE_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/checklist-templates/:templateId` | Delete a checklist template | Body: none | `DeleteResponse` | Template owner only | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TEMPLATE_NOT_FOUND` |

### Notes

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/notes` | List notes for a trip | Query: `{ limit?, cursor?, sort? }` | `NoteListResponse { data: NoteDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND` |
| `POST` | `/v1/trips/:tripId/notes` | Create a note | `{ title?, body }` | `NoteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `PATCH` | `/v1/trips/:tripId/notes/:noteId` | Update a note | `{ title?, body? }` | `NoteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 NOTE_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/notes/:noteId` | Delete a note | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 NOTE_NOT_FOUND` |

### Links

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/links` | List links for a trip | Query: `{ limit?, cursor?, sort? }` | `LinkListResponse { data: LinkDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND` |
| `POST` | `/v1/trips/:tripId/links` | Create a link | `{ title, url, description? }` | `LinkResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `PATCH` | `/v1/trips/:tripId/links/:linkId` | Update a link | `{ title?, url?, description? }` | `LinkResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 LINK_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/links/:linkId` | Delete a link | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 LINK_NOT_FOUND` |

### Budget Entries

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/budget-entries` | List budget entries and totals | Query: `{ category?, sort? }` | `BudgetEntryListResponse { data: BudgetEntryDto[], summary: { estimatedTotal, actualTotal } }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `POST` | `/v1/trips/:tripId/budget-entries` | Create a budget entry | `{ title, category, estimatedAmount?, actualAmount?, notes?, reservationId?, itineraryItemId? }` | `BudgetEntryResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `404 RELATED_RESOURCE_NOT_FOUND`, `409 BUDGET_REFERENCE_CONFLICT`, `422 VALIDATION_FAILED` |
| `PATCH` | `/v1/trips/:tripId/budget-entries/:entryId` | Update a budget entry | `{ title?, category?, estimatedAmount?, actualAmount?, notes?, reservationId?, itineraryItemId? }` | `BudgetEntryResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 ENTRY_NOT_FOUND`, `409 BUDGET_REFERENCE_CONFLICT`, `422 VALIDATION_FAILED` |
| `DELETE` | `/v1/trips/:tripId/budget-entries/:entryId` | Delete a budget entry | Body: none | `DeleteResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 ENTRY_NOT_FOUND` |

### Reminders

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/reminders` | List reminders for a trip | Query: `{ status?, targetType?, sort? }` | `ReminderListResponse { data: ReminderDto[] }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `POST` | `/v1/trips/:tripId/reminders` | Create and schedule a reminder | `{ recipientUserId, channel: "email", scheduledFor, offsetMinutes?, reservationId?, itineraryItemId?, checklistItemId? }` | `ReminderResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `404 RELATED_RESOURCE_NOT_FOUND`, `409 REMINDER_TARGET_CONFLICT`, `422 VALIDATION_FAILED`, `503 QUEUE_PUBLISH_FAILED` |
| `PATCH` | `/v1/trips/:tripId/reminders/:reminderId` | Update a scheduled reminder before dispatch | `{ scheduledFor?, offsetMinutes?, recipientUserId? }` | `ReminderResponse` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 REMINDER_NOT_FOUND`, `409 REMINDER_NOT_MUTABLE`, `422 VALIDATION_FAILED` |
| `POST` | `/v1/trips/:tripId/reminders/:reminderId/cancel` | Cancel a reminder without deleting its audit record | Body: none | `ReminderResponse { id, status: "cancelled", cancelledAt }` | Owner or editor | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 REMINDER_NOT_FOUND`, `409 REMINDER_ALREADY_DISPATCHED` |

### Notifications

| Method | Route | Purpose | Request body / query | Response shape | Authz rules | Error cases |
|---|---|---|---|---|---|---|
| `GET` | `/v1/trips/:tripId/notifications` | List reminder delivery records for a trip | Query: `{ limit?, cursor?, deliveryStatus?, sort? }` | `NotificationListResponse { data: NotificationDto[], pageInfo }` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 TRIP_NOT_FOUND`, `422 VALIDATION_FAILED` |
| `GET` | `/v1/trips/:tripId/notifications/:notificationId` | Get one delivery record | Path only | `NotificationResponse` | Active trip member | `401 UNAUTHENTICATED`, `403 FORBIDDEN`, `404 NOTIFICATION_NOT_FOUND` |

## MVP Endpoints Only

The MVP intentionally excludes:

- API-owned login, logout, password reset, and registration endpoints
- pending invitation accept/decline endpoints
- public share links
- attachment upload endpoints
- reservation import endpoints
- full-text search endpoints
- bulk mutation endpoints
- recurring reminder endpoints
- multi-recipient reminder fan-out in a single request
- notification resend endpoints exposed to end users
- internal-only operational endpoints from the public SPA surface

## Recommended Error Codes by Domain

- `UNAUTHENTICATED`
- `FORBIDDEN`
- `VALIDATION_FAILED`
- `DEPENDENCY_UNAVAILABLE`
- `EMAIL_CONFLICT`
- `TRIP_NOT_FOUND`
- `USER_NOT_FOUND`
- `RELATED_RESOURCE_NOT_FOUND`
- `TRIP_OR_MEMBER_NOT_FOUND`
- `TRIP_OR_TRAVELER_NOT_FOUND`
- `INVALID_DATE_RANGE`
- `ITEM_NOT_FOUND`
- `ITEM_REFERENCED_BY_REMINDER`
- `RESERVATION_NOT_FOUND`
- `RESERVATION_REFERENCED_BY_REMINDER`
- `CHECKLIST_NOT_FOUND`
- `CHECKLIST_REFERENCED_BY_REMINDER`
- `TEMPLATE_NOT_FOUND`
- `NOTE_NOT_FOUND`
- `LINK_NOT_FOUND`
- `ENTRY_NOT_FOUND`
- `REMINDER_NOT_FOUND`
- `NOTIFICATION_NOT_FOUND`
- `INVALID_TRIP_DATES`
- `INVALID_TRIP_STATE`
- `CANNOT_ADD_SECOND_OWNER`
- `OWNER_ROLE_REQUIRED`
- `CANNOT_CHANGE_LAST_OWNER`
- `CANNOT_REMOVE_LAST_OWNER`
- `TRAVELER_IN_USE`
- `BUDGET_REFERENCE_CONFLICT`
- `REMINDER_TARGET_CONFLICT`
- `REMINDER_NOT_MUTABLE`
- `REMINDER_ALREADY_DISPATCHED`
- `QUEUE_PUBLISH_FAILED`

## Final Design Notes

- Keep the API surface trip-centric so the frontend always operates inside a clear authorization boundary.
- Keep reminder creation synchronous from the client perspective, but keep delivery and retries queued.
- Keep list payloads lightweight for mobile and expose detail endpoints only where the UI needs them.
- Prefer additive changes in later iterations; do not broaden the public API beyond these MVP endpoints without a new design pass.
