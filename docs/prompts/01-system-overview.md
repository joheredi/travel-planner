<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/01-system-overview.md`.

Inputs:
- `docs/prd/travel-planner-prd.md`
- `docs/architecture/00-stack-decision.md`

Goal:
Design the overall system architecture for the MVP.

The document must include:
1. System context
   - primary users
   - external systems/services
2. Major runtime components
   - React SPA
   - NestJS API
   - PostgreSQL
   - reminder worker
   - Azure Service Bus
   - Azure Key Vault
   - Application Insights
3. High-level request flows
   - sign in
   - create trip
   - share trip
   - create reminder
   - process reminder
4. Bounded contexts / modules
   - identity & access
   - trip management
   - itinerary
   - reservations
   - checklists
   - notes & links
   - budget
   - reminders/notifications
5. Clear responsibility boundaries for each module
6. Recommended repo structure for apps and packages
7. Cross-cutting concerns
   - authn/authz
   - validation
   - error handling
   - telemetry
   - background jobs
8. Risks and simplifications for MVP

Include:
- one simple ASCII component diagram
- one section called “Constraints for downstream design docs”