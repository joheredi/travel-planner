## Title

Deliver collaborative overview panels and trip navigation polish

## ID

B020

## Milestone

M3-collaboration-and-sharing

## Objective

Complete the trip workspace summary experience by surfacing checklist progress, recent context, budget summary, and role-aware navigation.

## Background

After the collaboration and shared-content modules exist, the trip overview should reflect the full shared operational hub instead of just planning-core data. This is a bounded integration slice rather than a new domain.

## Scope

- Add collaborative overview summary panels for checklist progress, recent notes, recent links, and budget summary.
- Tighten trip-shell navigation so the shared workspace routes are easy to reach on mobile, including clearer section ordering and shared-trip context cues.
- Refine common role-aware trip-shell affordances now that all main modules exist.
- Keep aggregation logic lean and reusable.

## Out of Scope

- Reminder surfacing.
- Advanced analytics or activity history feeds.
- A redesign of the overall routing model.
- A global trip switcher or other new top-level navigation concept.

## Dependencies

- Blockers: `B013`, `B015`, `B016`, `B017`, `B018`, `B019`.
- Architecture refs: `docs/architecture/04-frontend-architecture.md`, `docs/prd/travel-planner-prd.md`, `docs/architecture/01-system-overview.md`.

## Target Files / Areas

- `apps/api/src/modules/trip-management/` or overview query services
- `apps/web/src/features/overview/`
- `apps/web/src/app/shells/TripShell.tsx`

## Implementation Notes

- Favor summary-oriented payloads over giant nested responses.
- Keep the overview and nav optimized for quick mobile access during planning and travel.
- Use role-aware affordances consistently but avoid duplicating backend policy rules.

## Acceptance Criteria

- The overview shows summary panels for checklist progress, recent notes, recent links, and budget totals.
- Trip navigation exposes all core trip modules cleanly on mobile with the collaborative modules added in `M3`.
- Users see role-appropriate overview actions and navigation cues.

## Tests Required

- Add frontend tests for overview summary rendering and role-aware navigation states.
- Add backend tests if new summary-query logic is added.

## Observability Requirements

- Rely on baseline request telemetry unless overview aggregation introduces non-trivial query paths needing explicit spans.

## Documentation Updates

- Update route or UX docs if they describe overview contents or trip navigation.

## Agent Notes

- This task is integration polish, not a reason to refactor every feature module.
- Keep performance in mind because the overview aggregates several modules.
