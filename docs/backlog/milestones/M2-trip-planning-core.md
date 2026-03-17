# M2 - Trip Planning Core

## Objective

Deliver the first genuinely useful planning workflow for a trip owner: define who is traveling, build the daily plan, and store booking details in a mobile-friendly trip workspace.

## In-scope capabilities

- traveler CRUD per trip, including dependent/minor flags and optional contact metadata
- itinerary item CRUD with:
  - date
  - optional times
  - category
  - chronological ordering
  - day-by-day views
- reservation CRUD with:
  - reservation type
  - provider and confirmation metadata
  - time and location fields
  - optional traveler associations
  - optional explicit itinerary linkage
- trip overview upgrades that summarize:
  - upcoming itinerary items
  - upcoming reservations
  - traveler counts
- responsive mobile-first UI for trip planning routes:
  - `/trips/:tripId/overview`
  - `/trips/:tripId/itinerary`
  - `/trips/:tripId/reservations`
- API support for:
  - travelers endpoints
  - itinerary endpoints
  - reservations endpoints
  - same-trip reference validation between reservations, itinerary items, travelers, and trips
- automated tests for ordering, validation, authorization, and cross-trip integrity on the new modules

## Implementation slices

1. extend Prisma schema for `Traveler`, `ItineraryItem`, `Reservation`, and reservation-traveler linkage
2. ship travelers API and UI first so reservation selectors have real data
3. ship itinerary CRUD and day grouping
4. ship reservations CRUD and optional itinerary linkage
5. wire overview summary panels from real trip data

## Out-of-scope items

- trip sharing UI and owner/editor/viewer collaboration flows
- checklist and checklist template features
- notes and links
- budget tracking
- reminder scheduling and delivery
- automatic itinerary creation from reservations
- import from confirmation emails or external booking sources

## Dependencies

- completed `M1-foundation`
- traveler, itinerary, and reservation boundaries from `docs/architecture/01-system-overview.md` and `docs/architecture/02-domain-model.md`
- route and page ownership from `docs/architecture/04-frontend-architecture.md`
- API contracts from `docs/architecture/03-api-design.md`
- test and observability expectations from `docs/architecture/08-observability-testing.md`

## Completion criteria

- trip owners can add, edit, and remove travelers from a trip
- trip owners can add, edit, reorder, and remove itinerary items, and the itinerary view remains chronologically sorted
- trip owners can add, edit, and remove reservations with optional traveler associations and optional itinerary links
- the trip overview surfaces upcoming itinerary and reservation information pulled from the real modules, not placeholders
- cross-trip references are rejected explicitly by the API
- reservation CRUD remains separate from itinerary CRUD; no implicit reservation-to-itinerary coupling is introduced
- mobile layouts for itinerary and reservation management are usable without desktop-only controls
- regression coverage exists for validation, ordering, and not-found/forbidden behavior on the new endpoints

## Risk notes

- The biggest risk is leaking cross-module coupling, especially between reservations and itinerary items. Keep them separate and link them only through explicit optional references.
- Sorting and date/time handling can get brittle quickly. Use stable ordering rules and trip timezone assumptions consistently.
- Traveler management is easy to underspecify. Keep travelers distinct from users and trip members so later collaboration work does not force model rewrites.
