<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/02-domain-model.md`.

Inputs:
- `docs/prd/travel-planner-prd.md`
- `docs/architecture/00-stack-decision.md`
- `docs/architecture/01-system-overview.md`

Goal:
Define the core domain model for the Travel Planner MVP.

The document must include:
1. Core entities with fields and purpose
   - User
   - Trip
   - TripMember
   - Traveler
   - ItineraryItem
   - Reservation
   - Checklist
   - ChecklistItem
   - ChecklistTemplate
   - Note
   - Link
   - BudgetEntry
   - Reminder
   - Notification
2. Entity relationships
3. Ownership and authorization implications
4. Lifecycle states
   - trip status
   - invitation status
   - reminder status
   - notification delivery status
5. Business invariants
   - examples: only owners can delete trips, viewers cannot mutate data, reservations belong to a trip, etc.
6. Domain events worth modeling
   - trip created
   - collaborator invited
   - reminder scheduled
   - reminder dispatched
7. Recommended IDs and timestamps conventions
8. Soft delete vs hard delete decisions
9. Fields that should be optional vs required
10. MVP simplifications

Include:
- a normalized conceptual schema section
- a section called “Rules agents must preserve during implementation”