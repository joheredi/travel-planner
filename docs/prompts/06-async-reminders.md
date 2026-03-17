<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/06-async-reminders.md`.

Inputs:
- `docs/architecture/01-system-overview.md`
- `docs/architecture/02-domain-model.md`
- `docs/architecture/03-api-design.md`
- `docs/architecture/05-auth-security.md`

Goal:
Design the reminders and notifications subsystem for the MVP.

The document must include:
1. What kinds of reminders exist in MVP
2. Which entities can create reminders
3. Scheduling model
4. Service Bus usage model
   - queue/topic choice
   - message shape
   - retry strategy
   - poison/dead-letter handling assumptions
5. Worker responsibilities in Azure Container Apps
6. Delivery channels
   - email now
   - in-app notification later if deferred
7. Idempotency and duplicate suppression
8. Failure handling and observability
9. Data model implications
10. MVP simplifications

Also include:
- one end-to-end reminder flow
- a section called “What remains synchronous in the API and what is queued”