<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/03-api-design.md`.

Inputs:
- `docs/architecture/00-stack-decision.md`
- `docs/architecture/01-system-overview.md`
- `docs/architecture/02-domain-model.md`

Goal:
Design the HTTP API for the MVP.

Requirements:
1. Use REST conventions suitable for NestJS
2. Define resource-oriented endpoints for:
   - auth/session integration assumptions
   - trips
   - trip members / sharing
   - travelers
   - itinerary items
   - reservations
   - checklists and checklist items
   - checklist templates
   - notes
   - links
   - budget entries
   - reminders
   - notifications
3. For each endpoint include:
   - method
   - route
   - purpose
   - request body shape
   - response shape
   - authz rules
   - error cases
4. Define pagination, filtering, and sorting strategy
5. Define validation approach
6. Define API error envelope
7. Define idempotency rules where relevant
8. Recommend OpenAPI generation strategy in NestJS

Also include:
- endpoint naming conventions
- DTO design guidance
- what should be synchronous vs queued
- “MVP endpoints only” section