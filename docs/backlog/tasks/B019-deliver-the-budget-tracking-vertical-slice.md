## Title

Deliver the budget tracking vertical slice

## ID

B019

## Milestone

M3-collaboration-and-sharing

## Objective

Implement trip-level budget entries with estimate-versus-actual tracking and category totals.

## Background

The PRD includes basic budget tracking but explicitly avoids advanced accounting. This task delivers that bounded scope and integrates it into the collaborative trip workspace.

## Scope

- Extend the schema with `BudgetEntry`.
- Implement budget-entry CRUD endpoints and trip-level summary totals.
- Add the `/trips/:tripId/budget` UI.
- Support optional linkage to reservations or itinerary items where helpful.

## Out of Scope

- Multi-currency conversion.
- Splitwise-style settlement or reimbursement workflows.
- Advanced reporting beyond trip totals and categories.

## Dependencies

- Blockers: `B011`, `B012`, `B014`.
- Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`.

## Target Files / Areas

- `packages/db/prisma/schema.prisma`
- `apps/api/src/modules/budget/`
- `apps/web/src/features/budget/`
- `packages/contracts/` budget DTOs

## Implementation Notes

- Use the trip currency as the only currency model in MVP.
- Allow linking to reservations or itinerary items, but at most one such link per budget entry.
- Keep totals simple and fast to compute.
- Own the budget route's feature-local role-aware controls here instead of waiting on broader UI polish from `B015`.

## Acceptance Criteria

- Owners and editors can create, edit, delete, and list budget entries.
- The budget screen shows estimated-versus-actual totals by trip and by category.
- Viewers can read budget data but cannot mutate it.
- The budget route shows feature-local read-only or disabled states consistent with owner/editor/viewer behavior.

## Tests Required

- Add API tests for CRUD, category totals, role enforcement, and invalid dual-reference behavior.
- Add frontend tests for budget forms, totals rendering, and viewer/editor/owner UI states.

## Observability Requirements

- Log budget mutations and failures with IDs and category metadata only; avoid logging budget notes by default.

## Documentation Updates

- Update API docs and budget route docs.
- Document the MVP single-currency constraint if needed in product/developer docs.

## Agent Notes

- Do not let this task expand into accounting features.
- Keep validation around linked resources explicit.
