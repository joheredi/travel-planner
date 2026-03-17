## Title

Provision the dev environment and GitHub Actions delivery baseline

## ID

B009

## Milestone

M1-foundation

## Objective

Create the minimum Azure and CI/CD baseline needed to build, validate, and deploy the MVP stack into the shared dev environment.

## Background

The deployment docs lock the MVP onto Azure Static Web Apps, Container Apps, PostgreSQL, Key Vault, and GitHub Actions with OIDC. This task lays down the deployable baseline without waiting for all feature work to finish.

## Scope

- Create the initial Bicep module structure and `dev` environment entrypoint.
- Provision the foundational Azure resources required for a shared-dev deployment baseline: Static Web App, Container Apps Environment, API Container App, worker Container App scaffold, PostgreSQL, Key Vault, and Application Insights / Log Analytics.
- Create GitHub Actions workflows for CI and dev deployment using OIDC.
- Wire API and worker runtime configuration to environment variables and Key Vault references where applicable.
- Make the expected local/shared-dev validation path explicit for Prisma migrations and shared-dev deployment smoke checks.

## Out of Scope

- Reminder-specific messaging and communications resources if they are not required for the initial scaffold.
- Staging and production promotion gates.
- Private networking or other deferred platform hardening.

## Dependencies

- Blockers: `B001`, `B002`, `B003`, `B004`.
- Architecture refs: `docs/architecture/07-azure-deployment.md`, `docs/architecture/00-stack-decision.md`, `docs/architecture/08-observability-testing.md`.

## Target Files / Areas

- `infra/bicep/`
- `.github/workflows/`
- `apps/api/.env.example`
- `apps/worker/.env.example`
- `apps/web/.env.example`

## Implementation Notes

- Match the recommended Bicep module boundaries closely.
- Use immutable build artifacts and OIDC authentication to Azure.
- Keep the worker deployable even if it only runs a scaffolded runtime at this stage.
- Treat this as one task with three explicit workstreams: CI checks, shared-dev cloud baseline, and runtime configuration wiring.

## Acceptance Criteria

- CI runs install, lint, type-check, tests, builds, Prisma validation, and Bicep validation.
- A shared-dev deployment path exists for the SPA and API, with the worker also deployable as a healthy runtime.
- The shared-dev Bicep baseline provisions the Azure resources needed for later API/worker telemetry and deployment validation.
- Required runtime configuration is documented and wired so API and worker secrets come from deployment configuration / Key Vault rather than checked-in files.
- No production secrets are stored in repository files or long-lived GitHub secrets when OIDC/Key Vault can avoid them.

## Tests Required

- Run workflow validation and Bicep validation locally or in CI.
- Smoke-verify a dev deployment if the environment is available.

## Observability Requirements

- Provision Application Insights / Log Analytics wiring required for later API and worker telemetry.
- Expose environment labels and configuration needed for trace and log attribution.

## Documentation Updates

- Document CI/CD prerequisites, Azure OIDC setup, and dev deployment expectations.

## Agent Notes

- Keep the dev topology aligned with the architecture docs; do not invent alternate hosting paths.
- Avoid coupling this task to reminder-specific runtime logic.
