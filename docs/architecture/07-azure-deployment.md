# Azure Deployment Model for Travel Planner MVP

## Purpose

This document defines the Azure deployment model for the Travel Planner MVP across local development, shared dev, test/staging, and production. It builds on:

- `docs/architecture/00-stack-decision.md`
- `docs/architecture/01-system-overview.md`
- `docs/architecture/05-auth-security.md`
- `docs/architecture/06-async-reminders.md`

The goal is to provide a deployable, cost-conscious Azure baseline that fits the locked stack and is specific enough for backlog, infrastructure, and delivery work to follow without reopening platform decisions.

## Deployment principles

- Keep the runtime split fixed: one SPA, one API, one worker.
- Keep environments separated at the Azure resource level.
- Keep production secrets in Key Vault and out of source control, GitHub secrets, and browser bundles.
- Prefer Azure managed identity for server-side access to Azure resources.
- Keep networking simple for MVP unless a concrete security or compliance requirement justifies private networking.
- Keep infrastructure declarative in Bicep.
- Keep rollout and rollback operationally boring: immutable build artifacts, revisioned container deploys, and manual promotion into production.

## Azure resources for MVP

The MVP cloud footprint should include these resources in each cloud environment unless otherwise noted:

- Azure Static Web Apps for the React SPA
- Azure Container Apps for:
  - the NestJS API
  - the NestJS worker
- Azure Container Apps Environment to host the API and worker apps
- Azure Database for PostgreSQL Flexible Server
- Azure Service Bus namespace with at least one queue:
  - `reminder-dispatch`
- Azure Key Vault
- Azure Monitor / Application Insights
- Log Analytics Workspace for Container Apps and centralized logs
- Azure Communication Services Email for reminder delivery

Recommended supporting resource:

- Azure Container Registry for versioned API and worker images

Why include Azure Container Registry:

- Container Apps deployments need a stable image source
- immutable image tags make rollback and promotion simpler
- it keeps backend deployment assets inside Azure

## Deployable resource inventory

### Resource inventory by cloud environment

Each cloud environment should have its own resource group and its own data-bearing resources.

| Resource | Shared dev | Test/staging | Production | Notes |
|---|---|---|---|---|
| Resource group | 1 | 1 | 1 | Separate resource group per environment |
| Static Web App | 1 | 1 | 1 | Public SPA host and environment-specific frontend settings |
| Container Apps Environment | 1 | 1 | 1 | Shared host for API and worker in that environment |
| Container App: API | 1 | 1 | 1 | Public HTTPS ingress enabled |
| Container App: Worker | 1 | 1 | 1 | No public ingress |
| PostgreSQL Flexible Server | 1 | 1 | 1 | Separate server per environment; no shared prod/non-prod database |
| Service Bus namespace | 1 | 1 | 1 | Queue-based reminder transport |
| Service Bus queue: `reminder-dispatch` | 1 | 1 | 1 | Scheduled reminder delivery and retries |
| Key Vault | 1 | 1 | 1 | Environment-scoped secrets only |
| Application Insights | 1 | 1 | 1 | App and worker telemetry |
| Log Analytics Workspace | 1 | 1 | 1 | Container Apps and platform logging |
| Azure Communication Services Email | 1 | 1 | 1 | Use sandbox or controlled sender in non-prod |
| Azure Container Registry | shared non-prod ACR | shared non-prod ACR | 1 dedicated prod ACR | Use one ACR shared by dev/staging and one separate ACR for prod |

### Local environment inventory

Local development is not an Azure deployment target. It should use:

- local Vite dev server for `apps/web`
- local NestJS API process
- local NestJS worker process
- local PostgreSQL via Docker Compose or equivalent containerized workflow
- local `.env` files using placeholders or non-production credentials

Local development should avoid mandatory Azure dependencies except when a shared dev integration test genuinely needs them.

## Environment strategy

### Local

Purpose:

- fast developer iteration
- feature work without cloud deployment friction
- local schema migrations and seeded test data

Rules:

- do not depend on Key Vault for local bootstrapping
- use app-scoped `.env.local` files that are never committed
- use a local Postgres container
- default the reminder delivery adapter to a safe non-destructive mode
- allow optional connection to shared dev Service Bus or other Azure resources only for integration testing

### Shared dev

Purpose:

- team integration environment
- default target for auto-deploy from the main branch
- realistic cloud validation without production cost

Rules:

- one shared environment is sufficient for MVP
- use small SKUs and autoscaling with conservative limits
- allow disposable or reseedable data
- use non-production Clerk configuration and email sender settings
- treat uptime as useful but not mission-critical

### Test/staging

Purpose:

- pre-production validation with production-like topology
- smoke testing of infrastructure, migrations, and deployment workflows
- final environment for release candidate verification

Rules:

- keep topology aligned with production even if scale is smaller
- use stable URLs and managed secrets
- protect access to staging admin surfaces and data
- use sanitized or synthetic data; do not clone production PII by default

### Production

Purpose:

- live customer traffic
- canonical operational environment

Rules:

- separate resource group from all non-prod environments
- preferably separate Azure subscription if available; if not, use strict RBAC separation in the same subscription
- production deployments require approval gates
- production data is never reused in lower environments
- custom domains, HTTPS, telemetry, backup retention, and operational alerting are mandatory

## Recommended environment topology

For MVP, use three cloud environments plus local:

- `local`
- `dev`
- `staging`
- `prod`

Recommended mapping:

- `dev` auto-deploys from `main`
- `staging` deploys from a manually promoted main commit or release branch/tag
- `prod` deploys only from an approved release artifact

This keeps cloud cost and process overhead reasonable while preserving one production-like checkpoint before release.

## Resource layout and ownership

### Resource groups

Use one resource group per cloud environment:

- `rg-travelplanner-dev`
- `rg-travelplanner-staging`
- `rg-travelplanner-prod`

This is the minimum useful isolation boundary for lifecycle management, RBAC, and cost reporting.

### Container Apps layout

Within each environment:

- one Container Apps Environment
- one API Container App
- one worker Container App

API Container App:

- public ingress enabled
- HTTPS only
- environment-specific CORS allowlist including the matching Static Web App origin
- scale based on HTTP load

Worker Container App:

- no public ingress
- scale based on Service Bus queue depth or equivalent event-driven rule
- idempotent reminder processing assumed, so brief overlap during revision replacement is acceptable

### Database layout

Use one PostgreSQL Flexible Server per cloud environment.

Rules:

- no shared server between production and non-production
- one application database per environment is sufficient for MVP
- use Prisma migrations as the only schema change path
- do not allow ad hoc schema drift between environments

### Messaging layout

Use one Service Bus namespace per cloud environment.

MVP queue inventory:

- `reminder-dispatch`

Deferred until needed:

- separate queues for dead-letter replay tools
- topic fan-out for multiple downstream consumers

### Observability layout

Use one Application Insights resource and one Log Analytics Workspace per cloud environment.

This keeps traces, logs, and metrics environment-scoped and avoids confusing cross-environment signal mixing.

## Configuration and secret flow

### Configuration categories

Split runtime configuration into three groups:

1. **Public frontend configuration**
   - values intentionally exposed to the browser
   - examples:
     - API base URL
     - Clerk publishable key
     - telemetry environment label

2. **Server application configuration**
   - non-secret runtime values used by API and worker
   - examples:
     - environment name
     - queue name
     - allowed frontend origin
     - telemetry sampling configuration

3. **Secrets**
   - sensitive values used only by server runtimes
   - examples:
     - Clerk secret key
     - PostgreSQL credentials
     - Azure Communication Services Email credentials when required
     - any raw connection strings retained for compatibility

### Local flow

Local development uses:

- `.env.example` files committed per app
- uncommitted `.env.local` files for actual values
- placeholder or local-only secrets

Rules:

- never copy production secrets into local files
- never use browser-exposed `VITE_` variables for secret material
- validate local server env vars at startup just like deployed environments

### Cloud flow

Preferred flow for `dev`, `staging`, and `prod`:

1. Bicep provisions the environment resources.
2. Bicep outputs non-secret resource identifiers and endpoints.
3. Secrets are written to the environment's Key Vault.
4. API and worker Container Apps receive non-secret config directly as environment variables.
5. API and worker resolve secrets from Key Vault through managed identity-backed references where supported.
6. Static Web Apps receives only browser-safe configuration values.

Rules:

- Key Vault is the only approved production secret source
- prefer managed identity over connection strings for Service Bus and Key Vault access
- keep PostgreSQL credentials in Key Vault until a later move to a stronger managed identity-compatible database auth model is justified
- do not store long-lived production secrets in GitHub repository secrets if OIDC and Key Vault can avoid it

### Secret owners by runtime

`apps/web`:

- may receive only public values such as:
  - `VITE_API_BASE_URL`
  - `VITE_CLERK_PUBLISHABLE_KEY`
  - `VITE_APP_ENV`

`apps/api`:

- receives non-secret config directly
- reads secrets from Key Vault or Key Vault-backed references
- uses managed identity for Azure resource access where possible

`apps/worker`:

- same pattern as API
- additionally requires reminder-delivery configuration and messaging settings

## CI/CD expectations

### Source control and pipeline assumptions

Use GitHub Actions as the CI/CD system for MVP.

Authentication to Azure should use GitHub OIDC federation, not stored Azure publish profiles or long-lived service principal secrets.

### Continuous integration expectations

Every pull request should run:

- install with `pnpm`
- lint
- type-check
- unit and integration tests
- build for `web`, `api`, and `worker`
- Prisma schema and migration checks
- Bicep validation or `what-if` style validation for changed infrastructure

PRs should not deploy production resources directly.

### Continuous delivery expectations

Recommended promotion flow:

1. Merge to `main`.
2. CI builds immutable artifacts:
   - frontend bundle
   - API image
   - worker image
3. CI publishes backend images to Azure Container Registry.
4. CI deploys `dev` automatically.
5. `staging` deployment requires an approval gate.
6. `prod` deployment requires an explicit approval gate after staging validation.

### Artifact strategy

Use immutable version identifiers for every deployment:

- git SHA for traceability
- semver or release tag for human-readable release labeling if introduced later

Do not deploy mutable `latest` tags to staging or production.

### Infrastructure deployment expectations

- keep Bicep in the same monorepo
- deploy infrastructure before application rollout when resources or settings changed
- separate infra and app deployment steps, but keep them in the same overall release workflow
- use idempotent Bicep deployments so repeated runs converge safely

## Networking assumptions at MVP level

The MVP should use a simple public-endpoint model with strong identity, TLS, and application-layer controls instead of full private networking.

### Public ingress assumptions

- Static Web Apps is public over HTTPS
- API Container App is public over HTTPS
- worker Container App has no public ingress

### Managed service access assumptions

- PostgreSQL Flexible Server may use public access with TLS enforced
- Service Bus may use its public endpoint with managed identity authorization where supported
- Key Vault may use its public endpoint with managed identity and environment-scoped RBAC
- Application Insights and Azure Monitor remain public Azure platform services

### MVP networking posture

- no VPN dependency for ordinary app access
- no private endpoints required in MVP
- no dedicated WAF, Front Door, or hub-and-spoke network required in MVP
- no NAT gateway requirement in MVP
- strict CORS configuration is still required for the API
- production uses custom domains and HTTPS

This is intentionally simple and cost-conscious. If later requirements demand stricter network isolation, the likely upgrade path is:

- VNet-integrated Container Apps Environment
- private access for PostgreSQL, Service Bus, and Key Vault
- controlled egress
- possibly Front Door or WAF at the edge

Those are explicitly deferred from the MVP baseline.

## Backup and restore expectations for PostgreSQL

Use PostgreSQL Flexible Server automated backups as the baseline recovery mechanism in all cloud environments.

### Backup expectations

`dev`:

- short retention is acceptable
- treat the environment as recoverable but partly disposable

`staging`:

- keep enough retention to validate restore workflows before production changes

`prod`:

- enable point-in-time restore capability with materially longer retention than non-prod
- retain backups long enough to cover normal operational rollback and incident investigation needs

Recommended MVP posture:

- `dev`: 7-day retention
- `staging`: 7-14 day retention
- `prod`: 21-35 day retention

### Restore expectations

- document the exact restore runbook for each environment
- perform at least occasional non-prod restore validation from production-like backup settings
- before risky or irreversible production schema changes, create an additional logical backup such as `pg_dump`
- treat restore time and data loss window as explicit operational concerns, not assumptions

### Migration safety rules

- Prisma migrations must run before app versions depend on new schema
- destructive migrations require explicit review
- production schema rollouts should favor backward-compatible application changes where practical

## Rollout strategy for frontend and backend

### Frontend rollout

Static Web Apps should use versioned build artifacts produced in CI.

Recommended strategy:

- auto-deploy to `dev`
- deploy to `staging` after approval
- deploy to `prod` after backend compatibility is confirmed

Frontend rollback is artifact-based:

- redeploy the previous known-good frontend build

### API rollout

Use Container Apps revisions for API deployment.

Recommended strategy:

- deploy a new immutable image revision
- run health checks and smoke tests
- in `dev`, move traffic immediately to the new revision
- in `staging`, validate the new revision before release approval
- in `prod`, switch traffic only after approval and health validation

For MVP, a full blue/green traffic split is optional. Single active revision with fast rollback is sufficient if the team keeps revisions healthy and immutable.

### Worker rollout

The worker has no public ingress, so rollout is revision replacement rather than traffic shifting.

Recommended strategy:

- deploy the new worker image revision
- let the new revision become active
- deactivate the old revision after health validation

Because queue consumption is at-least-once and reminder delivery is designed to be idempotent, brief overlap during worker rollout is acceptable.

### Release ordering

Preferred release order for backend-impacting changes:

1. infrastructure changes if required
2. database migrations
3. API deployment
4. worker deployment
5. frontend deployment

Reason:

- API and worker must be ready before the frontend depends on new behavior
- migrations should land before code that requires them
- frontend is usually easiest to roll back independently

## Cost-conscious simplifications for MVP

- use one shared dev environment instead of per-developer Azure stacks
- keep one staging environment rather than multiple long-lived test tiers
- use public endpoints instead of private networking
- use one API app and one worker app, not separate container apps per module
- use one Service Bus namespace and one queue for reminders
- use one PostgreSQL server per environment with one application database
- share one non-prod Azure Container Registry if needed to reduce idle cost
- avoid over-provisioning autoscale limits in dev and staging
- defer global traffic management, WAF, CDN complexity, and multi-region failover
- defer production-read replicas and high-availability expansions unless real load or RTO requirements justify them

These choices are deliberate MVP trade-offs, not accidental omissions.

## Recommended Bicep module boundaries

Keep Bicep modular enough to support isolated updates without turning infrastructure into a maze of tiny files.

Recommended module boundaries:

1. `environment-foundation`
   - tags
   - naming conventions
   - Log Analytics Workspace
   - Application Insights
   - Container Apps Environment

2. `static-web-app`
   - Static Web App
   - frontend app settings

3. `container-registry`
   - Azure Container Registry

4. `database`
   - PostgreSQL Flexible Server
   - database creation
   - backup retention settings
   - firewall/public access settings

5. `messaging`
   - Service Bus namespace
   - `reminder-dispatch` queue

6. `secrets`
   - Key Vault
   - secret placeholders or references
   - managed identity access policies or RBAC assignments

7. `communications`
   - Azure Communication Services Email resources and sender configuration

8. `api-app`
   - Container App for API
   - ingress
   - scaling
   - environment variables and secret references
   - managed identity bindings

9. `worker-app`
   - Container App for worker
   - no-ingress configuration
   - queue-driven scaling
   - environment variables and secret references
   - managed identity bindings

10. `role-assignments`
    - managed identity access to Key Vault
    - managed identity access to Service Bus
    - registry pull permissions

### Root deployment structure

Use one environment entrypoint per cloud environment that composes the shared modules with environment-specific parameters:

- `infra/bicep/dev/main.bicep`
- `infra/bicep/staging/main.bicep`
- `infra/bicep/prod/main.bicep`

This keeps the module graph shared while making environment intent explicit.

## Infrastructure decisions that backlog tasks must assume

- The SPA is deployed to Azure Static Web Apps.
- The API and worker are deployed as separate Azure Container Apps.
- Each cloud environment has its own resource group, PostgreSQL server, Service Bus namespace, Key Vault, Application Insights resource, and Container Apps Environment.
- The worker has no public ingress and consumes Service Bus queue messages asynchronously.
- PostgreSQL is the only system of record; no feature should depend on queue state as canonical data.
- Key Vault is the approved production secret source.
- GitHub Actions with OIDC is the expected CI/CD path.
- Bicep is the required infrastructure-as-code mechanism.
- Local development is primarily local-process and local-Postgres based, not a miniature Azure stack.
- MVP networking assumes public Azure endpoints with TLS and identity controls, not private networking.
- Production rollout assumes approval-gated promotion and rollback via immutable artifacts or container revisions.
- Backlog tasks should not assume per-developer cloud environments, Kubernetes, Front Door, private endpoints, read replicas, or multi-region deployment unless a later architecture decision adds them.

## Final design guidance

- Keep the Azure footprint small but environment-scoped.
- Keep deployment automation explicit, immutable, and approval-gated where it matters.
- Keep secrets in Key Vault and use managed identity whenever possible.
- Keep networking simple for MVP, but document the upgrade path for stricter isolation later.
- Keep infrastructure and application rollout aligned so the frontend never depends on undeployed backend or schema changes.
