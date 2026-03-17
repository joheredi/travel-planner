## Title

Configure shared TypeScript, ESLint, Prettier, and task pipelines

## ID

B002

## Milestone

M1-foundation

## Objective

Establish the shared code-quality and type-safety baseline used by all apps and packages.

## Background

The stack decision and quality strategy require consistent linting, formatting, type-checking, and task orchestration. This task turns those defaults into reusable packages and root scripts.

## Scope

- Create shared TS config and ESLint config packages consumed by apps and internal packages.
- Add Prettier configuration if the repo does not already include one.
- Wire root and package-level scripts for lint, type-check, test, and build.
- Add app-scoped `.env.example` files for `apps/web`, `apps/api`, and `apps/worker`.

## Out of Scope

- Feature-specific test suites.
- CI/CD workflow implementation beyond what is required to support local commands.
- Production secret values.

## Dependencies

- Blockers: `B001`.
- Architecture refs: `docs/architecture/00-stack-decision.md`, `docs/architecture/04-frontend-architecture.md`, `docs/architecture/08-observability-testing.md`.

## Target Files / Areas

- `packages/tsconfig/`
- `packages/eslint-config/`
- root `package.json` scripts
- `apps/web/.env.example`
- `apps/api/.env.example`
- `apps/worker/.env.example`

## Implementation Notes

- Keep config packages small and reusable.
- Use `tsc --noEmit` for type-check tasks.
- Do not add alternate package managers or duplicate lockfiles.

## Acceptance Criteria

- Each app/package can consume the shared TS and ESLint config.
- The repo exposes standard root scripts for lint, type-check, test, and build.
- Each runtime app has a checked-in `.env.example` with non-secret placeholders only.

## Tests Required

- Run lint and type-check commands successfully against the initialized workspace.
- Verify `.env.example` files do not contain secrets and match expected variable names.

## Observability Requirements

- No feature telemetry is required.
- Server apps should reserve obvious configuration slots for telemetry environment labels and OpenTelemetry settings.

## Documentation Updates

- Document shared developer commands and env file expectations where the repo keeps setup guidance.

## Agent Notes

- Avoid configuration churn; downstream tasks should inherit these defaults without bespoke overrides.
- Keep frontend-only and backend-only settings separated cleanly.
