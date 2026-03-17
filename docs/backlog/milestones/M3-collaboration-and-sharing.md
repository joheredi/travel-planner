# M3 - Collaboration and Sharing

## Objective

Turn the single-owner planning workspace into a shared trip hub by adding role-based collaboration and the remaining shared planning modules needed for preparation and coordination.

## In-scope capabilities

- trip member listing, add, role update, and revoke flows for existing users
- owner/editor/viewer enforcement across trip-scoped reads and mutations
- role-aware UI affordances for trip pages and trip settings
- checklist CRUD with checklist item completion state
- checklist template CRUD and template-to-trip instantiation
- notes CRUD for freeform trip context
- links CRUD for important external references
- budget entry CRUD with:
  - category
  - estimated amount
  - actual amount
  - optional links to reservation or itinerary items
  - trip-level totals
- trip overview expansion to include:
  - checklist progress
  - recent notes/links
  - budget summary
- sharing route and collaboration-focused settings flows:
  - `/trips/:tripId/sharing`
  - role-aware behavior across existing trip routes

## Implementation slices

1. add `TripMember` management endpoints and owner-only sharing workflows
2. establish the shared backend policy layer for owner/editor/viewer checks across existing trip modules
3. ship the sharing route and common shell-level role-aware affordances for existing routes
4. ship checklists and checklist templates, including their feature-local role-aware states
5. ship notes and links, including their feature-local role-aware states
6. ship budget entries and trip totals, including their feature-local role-aware states
7. finish overview and navigation states for a shared trip workspace

## Out-of-scope items

- pending invitation flows for users without accounts
- public share links
- advanced audit history UI
- reminder dispatch and notification delivery
- in-app notification center
- advanced accounting or multi-currency support
- real-time collaborative editing or chat-style coordination

## Dependencies

- completed `M1-foundation`
- completed `M2-trip-planning-core`
- membership and role invariants from `docs/architecture/02-domain-model.md`
- security and sharing rules from `docs/architecture/05-auth-security.md`
- page and authorization-aware UI rules from `docs/architecture/04-frontend-architecture.md`
- budget, checklist, notes, and links boundaries from `docs/architecture/01-system-overview.md` and the PRD

## Dependency notes for execution

- `M2` traveler, itinerary, reservation, and overview routes start from an owner-led baseline.
- `B014` establishes the shared backend membership and authorization foundation that all M3 feature slices depend on.
- `B015` is limited to the sharing route and common role-aware shell/navigation affordances for the existing M1/M2 routes.
- `B016`, `B018`, and `B019` do not need to wait for all of `B015` to finish; they own the feature-local role-aware affordances for their own routes and controls.
- `B020` should only start after the overview baseline from `B013` and the collaborative content slices are in place.

## Completion criteria

- trip owners can share a trip with an existing account by email and assign `editor` or `viewer`
- the API enforces owner/editor/viewer behavior consistently across trips, travelers, itinerary, reservations, checklists, notes, links, and budget entries
- editors can mutate ordinary trip content but cannot manage membership or delete the trip
- viewers can read trip data but cannot mutate trip content or reminders
- users can create trip checklists, create from templates, and track completion state per trip instance
- users can create and manage notes, links, and budget entries inside a trip
- trip overview reflects checklist progress, recent supporting context, and budget totals
- automated coverage exists for owner, editor, viewer, and non-member cases on the changed modules

## Risk notes

- This milestone can sprawl if every screen invents its own permission logic. Centralize authorization checks and keep frontend role gating as affordance only.
- Sharing by email is security-sensitive. Normalize email consistently and preserve the invariant that the MVP only shares with existing accounts.
- Budget can drift into accounting complexity. Keep it to trip-level entries, category totals, and estimate-versus-actual comparisons only.
