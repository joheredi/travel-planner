# Travel Planner MVP Backlog

## Project Summary

This backlog translates the PRD, architecture set, and milestone plan into bounded implementation tasks for autonomous AI agents. Tasks are intentionally small enough for one focused work stream while still covering the full MVP.

## Milestones

- `M1-foundation` - Repository, platform, auth, trip ownership, and dev deployment baseline.
- `M2-trip-planning-core` - Travelers, itinerary, reservations, and planning overview flows.
- `M3-collaboration-and-sharing` - Sharing, role-aware UX, checklists, notes, links, budget, and collaborative overview polish.
- `M4-reminders-and-ops` - Reminders, async delivery, observability, release gates, and operational readiness.

## Task Index

Dependency note:

- `Blockers` in the summaries below are hard prerequisites.
- Soft sequencing guidance and rollout notes live in the individual task files and milestone docs.

| Status | ID | Title | Milestone | Dependency Summary |
|---|---|---|---|---|
| [ ] | `B001` | Scaffold the pnpm and Turborepo monorepo layout | `M1-foundation` | Blockers: none.; Architecture refs: `docs/architecture/00-stack-decision.md`, `docs/architecture/01-system-overview.md`, `docs/architecture/07-azure-deployment.md`. |
| [ ] | `B002` | Configure shared TypeScript, ESLint, Prettier, and task pipelines | `M1-foundation` | Blockers: `B001`.; Architecture refs: `docs/architecture/00-stack-decision.md`, `docs/architecture/04-frontend-architecture.md`, `docs/architecture/08-observability-testing.md`. |
| [ ] | `B003` | Create the Prisma database package and core persistence schema | `M1-foundation` | Blockers: `B001`, `B002`.; Architecture refs: `docs/architecture/02-domain-model.md`, `docs/architecture/07-azure-deployment.md`, `docs/architecture/08-observability-testing.md`. |
| [ ] | `B004` | Bootstrap the NestJS API foundation and contract conventions | `M1-foundation` | Blockers: `B001`, `B002`, `B003`.; Architecture refs: `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`, `docs/architecture/08-observability-testing.md`. Includes the shared API envelope/error contract baseline for later slices. |
| [ ] | `B005` | Deliver the Clerk session sync authentication slice | `M1-foundation` | Blockers: `B003`, `B004`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/05-auth-security.md`, `docs/architecture/04-frontend-architecture.md`. |
| [ ] | `B006` | Deliver the current-user profile management slice | `M1-foundation` | Blockers: `B005`.; Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`, `docs/architecture/05-auth-security.md`. |
| [ ] | `B007` | Deliver the trip CRUD and owner membership backend slice | `M1-foundation` | Blockers: `B003`, `B004`, `B005`.; Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`. |
| [ ] | `B008` | Deliver the trips shell and owner trip management UI | `M1-foundation` | Blockers: `B005`, `B007`.; Architecture refs: `docs/architecture/04-frontend-architecture.md`, `docs/architecture/03-api-design.md`, `docs/prd/travel-planner-prd.md`. |
| [ ] | `B009` | Provision the dev environment and GitHub Actions delivery baseline | `M1-foundation` | Blockers: `B001`, `B002`, `B003`, `B004`.; Architecture refs: `docs/architecture/07-azure-deployment.md`, `docs/architecture/00-stack-decision.md`, `docs/architecture/08-observability-testing.md`. |
| [ ] | `B010` | Deliver the traveler management vertical slice | `M2-trip-planning-core` | Blockers: `B007`, `B008`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`. |
| [ ] | `B011` | Deliver the itinerary management vertical slice | `M2-trip-planning-core` | Blockers: `B007`, `B008`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`. |
| [ ] | `B012` | Deliver the reservation management vertical slice | `M2-trip-planning-core` | Blockers: `B010`, `B011`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`. |
| [ ] | `B013` | Deliver the planning overview summary slice | `M2-trip-planning-core` | Blockers: `B010`, `B011`, `B012`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/04-frontend-architecture.md`, `docs/prd/travel-planner-prd.md`. |
| [ ] | `B014` | Deliver trip sharing and membership management on the backend | `M3-collaboration-and-sharing` | Blockers: `B005`, `B007`.; Architecture refs: `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`. |
| [ ] | `B015` | Deliver the sharing UI and role-aware frontend affordances | `M3-collaboration-and-sharing` | Blockers: `B014`, `B008`, `B013`.; Architecture refs: `docs/architecture/04-frontend-architecture.md`, `docs/architecture/05-auth-security.md`, `docs/prd/travel-planner-prd.md`. |
| [ ] | `B016` | Deliver the checklist management vertical slice | `M3-collaboration-and-sharing` | Blockers: `B014`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`. Includes feature-local role-aware UI states for checklist routes. |
| [ ] | `B017` | Deliver the checklist template vertical slice | `M3-collaboration-and-sharing` | Blockers: `B006`, `B016`.; Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`. |
| [ ] | `B018` | Deliver the notes and links vertical slice | `M3-collaboration-and-sharing` | Blockers: `B014`.; Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/01-system-overview.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`. Includes feature-local role-aware UI states for notes and links routes. |
| [ ] | `B019` | Deliver the budget tracking vertical slice | `M3-collaboration-and-sharing` | Blockers: `B011`, `B012`, `B014`.; Architecture refs: `docs/prd/travel-planner-prd.md`, `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/04-frontend-architecture.md`. Includes feature-local role-aware UI states for budget routes. |
| [ ] | `B020` | Deliver collaborative overview panels and trip navigation polish | `M3-collaboration-and-sharing` | Blockers: `B013`, `B015`, `B016`, `B017`, `B018`, `B019`.; Architecture refs: `docs/architecture/04-frontend-architecture.md`, `docs/prd/travel-planner-prd.md`, `docs/architecture/01-system-overview.md`. Scope is limited to checklist, notes/links, and budget overview panels plus trip-shell navigation affordances. |
| [ ] | `B021` | Deliver the reminder data model and API vertical slice | `M4-reminders-and-ops` | Blockers: `B011`, `B012`, `B014`, `B016`.; Architecture refs: `docs/architecture/02-domain-model.md`, `docs/architecture/03-api-design.md`, `docs/architecture/05-auth-security.md`, `docs/architecture/06-async-reminders.md`. Covers reminder API support for itinerary items, reservations, and checklist items. |
| [ ] | `B022` | Deliver Service Bus scheduling and worker dispatch for reminders | `M4-reminders-and-ops` | Blockers: `B009`, `B021`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/06-async-reminders.md`, `docs/architecture/07-azure-deployment.md`. |
| [ ] | `B023` | Deliver email reminder rendering and notification history | `M4-reminders-and-ops` | Blockers: `B021`, `B022`.; Architecture refs: `docs/architecture/01-system-overview.md`, `docs/architecture/06-async-reminders.md`, `docs/architecture/07-azure-deployment.md`. |
| [ ] | `B024` | Deliver reminder observability dashboards and alerts | `M4-reminders-and-ops` | Blockers: `B009`, `B022`, `B023`.; Architecture refs: `docs/architecture/06-async-reminders.md`, `docs/architecture/07-azure-deployment.md`, `docs/architecture/08-observability-testing.md`. |
| [ ] | `B025` | Deliver release gates, rollout workflow, and operational runbooks | `M4-reminders-and-ops` | Blockers: `B009`, `B023`, `B024`.; Architecture refs: `docs/architecture/07-azure-deployment.md`, `docs/architecture/08-observability-testing.md`. |
