## Title

Scaffold the pnpm and Turborepo monorepo layout

## ID

B001

## Milestone

M1-foundation

## Objective

Create the repository structure and workspace wiring for `apps/web`, `apps/api`, `apps/worker`, and the initial shared packages so later feature tasks can land in stable locations.

## Background

The stack and system docs lock the repository around a pnpm workspace, Turborepo, three runtime apps, and a small set of shared packages. This task establishes that skeleton without pulling feature logic forward.

## Scope

- Create or normalize the root workspace manifests: `package.json`, `pnpm-workspace.yaml`, and `turbo.json`.
- Create the baseline directory structure for `apps/web`, `apps/api`, `apps/worker`, `packages/db`, `packages/contracts`, `packages/ui`, `packages/eslint-config`, and `packages/tsconfig`.
- Add placeholder package manifests and scripts so the workspace can install and run top-level tasks.
- Add consistent naming for apps and packages to match the architecture docs.

## Out of Scope

- Implementing business features.
- Provisioning Azure resources.
- Final ESLint, testing, or app bootstrap logic beyond what is needed to make the workspace coherent.

## Dependencies

- Blockers: none.
- Architecture refs: `docs/architecture/00-stack-decision.md`, `docs/architecture/01-system-overview.md`, `docs/architecture/07-azure-deployment.md`.

## Target Files / Areas

- `package.json`
- `pnpm-workspace.yaml`
- `turbo.json`
- `apps/web/`
- `apps/api/`
- `apps/worker/`
- `packages/db/`
- `packages/contracts/`
- `packages/ui/`
- `packages/eslint-config/`
- `packages/tsconfig/`

## Implementation Notes

- Follow the repo layout from the system overview exactly; do not invent extra top-level packages.
- Keep package scripts minimal but real so downstream tasks can extend them instead of replacing them.
- Prefer boring workspace conventions over custom bootstrap wrappers.

## Acceptance Criteria

- A fresh `pnpm install` succeeds at the repository root.
- The workspace contains the expected apps and packages in the expected locations.
- Top-level Turborepo task wiring exists for build, lint, test, and type-check, even if some tasks are placeholder implementations initially.

## Tests Required

- Run `pnpm install` successfully.
- Run the root Turborepo command that enumerates configured tasks without workspace resolution errors.

## Observability Requirements

- No production telemetry is required yet.
- Leave the app/package layout compatible with later OpenTelemetry and logging setup.

## Documentation Updates

- Update any root README or developer bootstrap instructions if the repo starts including them in this task.

## Agent Notes

- Do not add speculative packages such as `core` or `shared-utils`.
- This task should leave later tasks with a clean place to land code, not a fully configured product.
