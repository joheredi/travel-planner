<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/07-azure-deployment.md`.

Inputs:
- `docs/architecture/00-stack-decision.md`
- `docs/architecture/01-system-overview.md`
- `docs/architecture/05-auth-security.md`
- `docs/architecture/06-async-reminders.md`

Goal:
Design the Azure deployment model for dev, test, and prod.

Requirements:
1. Define Azure resources for MVP
   - Static Web Apps
   - Container Apps
   - PostgreSQL Flexible Server
   - Service Bus
   - Key Vault
   - Application Insights / Azure Monitor
2. Define environment strategy
   - local
   - shared dev
   - test/staging
   - production
3. Define configuration and secret flow
4. Define CI/CD expectations
5. Define networking assumptions at MVP level
6. Define backup and restore expectations for Postgres
7. Define rollout strategy for frontend and backend
8. Define cost-conscious simplifications for MVP
9. Recommend Bicep module boundaries

Also include:
- a deployable resource inventory
- a section called “Infrastructure decisions that backlog tasks must assume”