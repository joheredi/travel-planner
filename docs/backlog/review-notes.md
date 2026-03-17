# Backlog Review Notes

## What changed

- clarified hard dependency semantics in `docs/backlog/index.md`
- fixed concrete blocker bugs in the task index:
  - `B009` now depends on `B003`
  - `B016`, `B018`, and `B019` no longer wait on `B015`
  - `B020` now depends on `B013`
  - `B021` now depends on `B011`, `B012`, `B014`, and `B016`
  - `B024` now depends on `B009`
- added a downstream-readiness section to `docs/backlog/milestones/M1-foundation.md` so later milestones know exactly what M1 must deliver
- clarified `M3` execution boundaries so sharing-shell work (`B015`) does not block feature-local collaboration slices unnecessarily
- clarified `M4` rollout ordering so reminder schema, scheduling, delivery, and ops hardening happen in the intended sequence
- tightened `B004` to explicitly own the shared API contract baseline and OpenAPI generation setup
- tightened `B009` so its scope is clearer across CI, shared-dev Azure baseline, and runtime configuration wiring
- narrowed `B015` to the sharing route plus common shell-level role-aware affordances for existing routes
- moved feature-local role-aware UI responsibility into `B016`, `B018`, and `B019`
- clarified `B020` so "navigation polish" now means collaborative overview panels and trip-shell navigation affordances, not an undefined redesign
- clarified `B021` so reminder support covers all three MVP target types and uses explicit async-safe fields
- clarified `B024` so instrumentation is expected to land in feature work first, with this task validating coverage and adding dashboards/alerts

## Remaining risks

- `B009`, `B025`, and to a lesser extent `B004` are still comparatively large tasks even after the scope cleanup; they are clearer now, but they may still be better executed as tightly scoped sub-steps inside implementation
- several tasks still have room for more precise acceptance criteria around limits, ordering rules, and edge-case validation; the biggest remaining examples are traveler cardinality, itinerary ordering details, and budget calculation rules
- reminder work remains operationally risky even with the dependency fixes; partial rollout or missing correlation data would still create hard-to-debug failures

## Tasks that still require human judgment

- whether to formally split large tasks such as `B009` and `B025` into new backlog items or keep them as single tasks with internal workstreams
- whether `B015` should remain a single task with narrowed scope or be split later into sharing-route work and broad UI-retrofit work if implementation pressure grows
- how strict the product should be on feature limits and validation details that the PRD leaves open, such as traveler count limits, note/link pagination caps, and budget entry constraints beyond the current architecture baseline
