# Frontend Architecture for Travel Planner MVP

## Purpose

This document defines the frontend architecture for the Travel Planner MVP SPA in `apps/web`. It builds on:

- `docs/architecture/00-stack-decision.md`
- `docs/architecture/01-system-overview.md`
- `docs/architecture/03-api-design.md`

The goal is to keep the React + Vite frontend simple, mobile-first, and consistent with the product's trip-centric domain model. The frontend is responsible for presentation, client-side routing, local interaction state, and API integration. It is not responsible for business authorization, direct data persistence, or reminder execution.

## Frontend Architectural Principles

- Build the product as a React + TypeScript + Vite SPA in `apps/web`.
- Treat the NestJS API as the source of business state and mutation rules.
- Treat Clerk as the source of identity and session state.
- Keep the UI trip-centric so most authenticated screens operate within a selected `tripId`.
- Prefer route-level data loading boundaries and feature-local components over a large shared global state layer.
- Optimize for mobile-first use, clear CRUD flows, and reliable loading and error handling.
- Use shared frontend primitives and contracts where reuse is real, but avoid speculative abstraction.

## App Shell and Route Structure

### Shell model

The SPA should use three shell layers:

1. `PublicShell`
   - used for sign-in, sign-up redirect surfaces, and any unauthenticated entry messaging
   - mostly delegates UI to Clerk components

2. `AuthenticatedShell`
   - wraps all signed-in routes
   - owns session bootstrap, top-level error boundary, and global navigation entry points
   - includes a small mobile-friendly header with trip switcher access, profile menu, and primary navigation trigger

3. `TripShell`
   - wraps trip-scoped routes under `/trips/:tripId/*`
   - owns the trip header, section navigation, role-aware action affordances, and common query boundaries for selected-trip context

This separation keeps unauthenticated auth flows independent from the main app shell and prevents trip-specific chrome from leaking into the trips list or account settings screens.

### Recommended route map

```text
/
  -> redirect to /trips when authenticated
  -> redirect to /sign-in when unauthenticated

/sign-in
/sign-up

/app
  /profile

/trips
  /new
  /:tripId
    /overview
    /itinerary
    /reservations
    /checklists
    /notes
    /links
    /budget
    /settings
    /sharing
```

Routing rules:

- Use `/trips` as the main authenticated landing route.
- Redirect `/trips/:tripId` to `/trips/:tripId/overview`.
- Keep notes and links as separate child routes even if they are visually grouped in navigation.
- Keep sharing and trip settings as separate routes because their permissions and failure states differ.
- Keep profile settings outside trip scope under `/app/profile`.

### Route ownership

- Public routes own Clerk-driven auth surfaces only.
- `/trips` owns trip discovery and trip creation entry points.
- `/trips/:tripId/*` owns all collaboration and planning modules tied to one trip.
- Trip routes should lazy-load by section to keep initial mobile load smaller.

## Recommended Frontend Libraries

The frontend should use a deliberately small library set with clear roles.

### Routing

- `react-router-dom`

Why:

- best fit for a Vite SPA
- mature nested routing model for `AuthenticatedShell` and `TripShell`
- straightforward route-level lazy loading and error boundaries
- no need for a full-stack React framework because rendering remains client-side

### Server state and data fetching

- `@tanstack/react-query`

Why:

- clear model for query caching, invalidation, background refetching, and mutation status
- reduces custom loading/error boilerplate for CRUD-heavy screens
- works well with mobile networks where stale-while-revalidate behavior improves perceived performance
- fits the API's list/detail/mutation patterns and cursor-based envelopes

### Forms

- `react-hook-form`

Why:

- low re-render overhead for mobile devices
- good ergonomics for nested form UIs and reusable field components
- integrates cleanly with Zod validation
- supports optimistic UI around submit buttons and field-level errors without custom form state plumbing

### Validation

- `zod`

Why:

- strong TypeScript inference
- suitable for frontend form schemas and response parsing at the client boundary
- can reuse contract-adjacent schema ideas from `packages/contracts` without coupling the UI directly to backend DTO classes

Validation rule:

- use shared schemas from `packages/contracts` when they describe transport-level inputs cleanly
- extend or narrow them in the frontend when presentation needs differ, such as converting empty strings to `undefined` before submit

### UI primitives

- `@radix-ui/react-*` primitives wrapped in `packages/ui`

Why:

- accessible, composable primitives for dialogs, dropdown menus, tabs, popovers, and scroll areas
- keeps `apps/web` focused on feature composition instead of low-level interaction behavior
- aligns with the existing repo direction that `packages/ui` should contain reusable presentational components only

Recommended supporting utilities:

- `clsx` for conditional class composition
- `tailwind-merge` only if a utility-class styling approach is adopted

The architecture should avoid bringing in a heavy component framework that would fight the product's small, custom information architecture.

## Suggested Frontend Directory Structure

```text
apps/
  web/
    src/
      app/
        App.tsx
        providers/
          AppProviders.tsx
          QueryProvider.tsx
          ClerkProvider.tsx
        router/
          index.tsx
          protected-route.tsx
        shells/
          PublicShell.tsx
          AuthenticatedShell.tsx
          TripShell.tsx
      features/
        auth/
        trips/
          api/
          components/
          hooks/
          pages/
          schemas/
        overview/
        itinerary/
        reservations/
        checklists/
        notes/
        links/
        budget/
        sharing/
        settings/
      components/
        layout/
        feedback/
        forms/
      lib/
        api/
          client.ts
          errors.ts
          auth.ts
          query-keys.ts
        formatting/
        routing/
        utils/
      styles/
      main.tsx
```

Directory rules:

- keep feature code inside `features/<feature-name>/`
- keep feature pages, hooks, schemas, and API adapters together
- keep `lib/api` focused on cross-feature client concerns, not domain-specific fetchers
- keep `components/` for app-wide shared composition pieces that are specific to the web app
- put reusable presentational primitives in `packages/ui`, not in `apps/web/src/components`
- do not create generic buckets like `shared`, `core`, or `helpers` unless a concrete boundary emerges

## Page Map for MVP

### Trips list

Route:

- `/trips`

Purpose:

- list trips visible to the current user
- surface trip status, dates, destination summary, and role summary
- provide entry points for create trip and open trip

Primary data:

- `GET /v1/trips`
- `POST /v1/session/sync` during authenticated bootstrap if needed

Primary actions:

- create trip
- filter by status or date
- open trip

### Trip overview

Route:

- `/trips/:tripId/overview`

Purpose:

- summary landing page for a selected trip
- show trip metadata, membership role, high-level counts, upcoming itinerary items, upcoming reservations, checklist progress, and budget summary

Primary data:

- `GET /v1/trips/:tripId`
- lightweight secondary queries for a few summary panels where the trip detail payload is not enough

Primary actions:

- edit trip metadata if role permits
- jump to itinerary, reservations, checklists, budget, sharing

### Itinerary

Route:

- `/trips/:tripId/itinerary`

Purpose:

- manage day-by-day itinerary items in chronological order

Primary data:

- `GET /v1/trips/:tripId/itinerary-items`

Primary actions:

- create, edit, delete itinerary items
- filter by date range or category

### Reservations

Route:

- `/trips/:tripId/reservations`

Purpose:

- manage booking records with traveler associations and optional itinerary linkage

Primary data:

- `GET /v1/trips/:tripId/reservations`
- `GET /v1/trips/:tripId/travelers` for traveler selectors
- itinerary item lookup only where linking UI is present

Primary actions:

- create, edit, delete reservations
- filter by type or dates

### Checklists

Route:

- `/trips/:tripId/checklists`

Purpose:

- manage trip checklists and checklist items
- support quick completion toggles and checklist-template flows

Primary data:

- `GET /v1/trips/:tripId/checklists`
- `GET /v1/checklist-templates`
- `GET /v1/trips/:tripId/checklists/:checklistId` for expanded detail

Primary actions:

- create checklist
- create from template
- add item
- complete item
- edit or delete checklist and items

### Notes and links

Routes:

- `/trips/:tripId/notes`
- `/trips/:tripId/links`

Purpose:

- keep lightweight collaborative reference material easy to capture on mobile

Primary data:

- `GET /v1/trips/:tripId/notes`
- `GET /v1/trips/:tripId/links`

Primary actions:

- create, edit, delete notes
- create, edit, delete links

### Budget

Route:

- `/trips/:tripId/budget`

Purpose:

- manage budget entries and show trip-level estimated versus actual totals

Primary data:

- `GET /v1/trips/:tripId/budget-entries`

Primary actions:

- create, edit, delete budget entries
- filter by category

### Sharing and settings

Routes:

- `/trips/:tripId/sharing`
- `/trips/:tripId/settings`
- `/app/profile`

Purpose:

- sharing handles trip membership changes and role management
- settings handles trip metadata updates, archival, and deletion
- profile handles current-user profile preferences

Primary data:

- `GET /v1/trips/:tripId/members`
- `GET /v1/trips/:tripId`
- `GET /v1/me`

Primary actions:

- add or remove member
- change role
- update trip metadata
- archive or delete trip if authorized
- update user profile

## Component Layering

The frontend should follow a narrow layering model to keep business complexity out of reusable UI.

### 1. App shell layer

Responsibilities:

- providers
- routing
- auth gates
- error boundaries
- shared layout chrome
- trip selection context from the active route

Examples:

- `AuthenticatedShell`
- `TripShell`
- app header and section nav
- route-level suspense and error boundaries

### 2. Page layer

Responsibilities:

- orchestrate feature queries and mutations
- map route params to feature hooks
- compose page sections and action bars
- decide which empty, loading, and error states to show

Pages should be thin orchestration components, not places where transport details or reusable field logic accumulate.

### 3. Feature components layer

Responsibilities:

- module-specific UI, such as trip cards, itinerary timeline groups, reservation forms, checklist item rows, budget summary panels, and member-role tables
- feature-specific hooks for query composition
- mutation wrappers tied to one domain area

These components may know domain language, but they should not own cross-feature shell concerns.

### 4. Shared UI layer

Responsibilities:

- buttons, inputs, dialogs, drawers, tabs, toasts, badges, cards, list containers, skeletons
- reusable, presentation-oriented building blocks with no backend logic

Location:

- shared app-specific composition components in `apps/web/src/components`
- low-level reusable primitives in `packages/ui`

### 5. Hooks and services layer

Responsibilities:

- API client wrappers
- query key factories
- transport-to-view-model mapping where needed
- URL state helpers
- formatting helpers for dates, money, and status labels

Rules:

- keep API functions close to their feature when they are feature-specific
- keep global client concerns centralized in `lib/api`
- do not hide every fetch in a generic service layer; prefer explicit feature adapters

## API Client Strategy

### Client structure

Use a thin typed API client built around `fetch` in `lib/api/client.ts`.

Responsibilities:

- set the `/v1` base path
- attach `Authorization: Bearer <Clerk JWT>` for authenticated requests
- normalize JSON parsing and request headers
- convert non-2xx responses into a typed `ApiError`
- expose typed `get`, `post`, `patch`, and `delete` helpers

Avoid introducing Axios unless a clear requirement emerges. Native `fetch` is sufficient for the MVP and keeps the dependency surface smaller.

### Error handling model

The client should parse the documented API error envelope:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request body failed validation.",
    "status": 422,
    "requestId": "req_123",
    "details": []
  }
}
```

Client rules:

- preserve `code`, `message`, `status`, and `requestId`
- expose validation `details` so forms can map field errors precisely
- treat `401` as a session/auth boundary issue
- treat `403` as a permissions issue and show role-aware messaging
- treat `404` inside trip scope as either missing or inaccessible content without leaking authorization details
- treat `503` as a retryable operational issue and show non-destructive retry UI

### Query and mutation conventions

- define query key factories by feature and resource ID
- invalidate the narrowest relevant query set after mutation
- prefer optimistic updates only for low-risk, easily reversible interactions such as checklist item completion
- prefer server-confirmed updates for destructive or role-sensitive actions
- keep trip-scoped queries keyed by `tripId` to prevent cache bleeding between trips

### Session bootstrap

At authenticated app startup:

1. Clerk resolves the user session.
2. The SPA obtains a token getter from Clerk.
3. The app calls `POST /v1/session/sync` once per fresh authenticated app bootstrap.
4. The result seeds current-user and membership-summary queries.

This flow keeps the frontend aligned with the API's local user bootstrap assumptions.

## Form Patterns

### General form rules

- use `react-hook-form` for all non-trivial create and edit forms
- validate at the form boundary with Zod
- submit normalized payloads that match API DTOs
- keep server validation messages visible even when client validation already exists
- disable submit while a mutation is in flight, but preserve form values and field errors

### Form composition

Preferred structure:

- page-level container handles query and mutation wiring
- feature form component owns fields and submit mapping
- shared field primitives live in `packages/ui` or `components/forms`

### Specific form guidance by domain

- trip forms should validate date range, timezone, and currency before submit
- reservation forms should support optional itinerary and traveler associations without forcing those choices
- checklist creation should support both blank and template-backed flows
- budget forms should accept decimal strings and format them consistently before submit
- sharing forms should normalize email input and handle role selection explicitly

### Mutation UX

- use inline field errors for validation failures
- use page-level or dialog-level alerts for conflict or dependency errors
- use confirmation dialogs for destructive actions such as deleting a trip, reservation, or member
- keep idempotency-sensitive actions safe from double submit by disabling buttons while pending

## Loading, Error, and Empty States Strategy

### Loading states

- use page-level skeletons for initial route loads
- use sectional skeletons for overview panels and tabular subregions
- use inline spinner states for small actions such as item completion or row-level saves
- avoid full-screen blocking loaders after the initial route bootstrap unless auth state is unresolved

### Error states

- use route-level error boundaries for unrecoverable page load failures
- use inline retry surfaces for query failures within a page section
- show structured form errors close to inputs when the API returns validation details
- use toast or banner feedback for successful mutations and non-field operational failures

### Empty states

Each core page should define an intentional empty state:

- trips list: no trips yet, with create-trip CTA
- itinerary: no itinerary items yet, with add-item CTA
- reservations: no reservations yet, with add-reservation CTA
- checklists: no checklists yet, with create or use-template CTAs
- notes: no notes yet, with add-note CTA
- links: no saved links yet, with add-link CTA
- budget: no budget entries yet, with add-entry CTA
- sharing: no collaborators yet, with share-trip CTA for owners

Empty states should explain the next useful action and avoid implying system failure.

## Mobile-First UX Rules

- Design every page for a narrow viewport first, then scale up to tablet and desktop.
- Keep the primary trip section navigation reachable with one thumb, preferably through a bottom sheet, tab bar, or compact drawer pattern.
- Keep top-level actions short and explicit: `Add trip`, `Add reservation`, `Share trip`.
- Prefer stacked cards, grouped sections, and progressive disclosure over dense desktop tables.
- Do not require side-by-side form layouts on mobile.
- Make touch targets comfortably tappable and preserve spacing between destructive and non-destructive actions.
- Keep important summary information above the fold on the trip overview page.
- Use sticky save bars or sticky action footers only for forms where the action might otherwise scroll out of reach.
- Use route-level code splitting and query caching to keep repeated navigation responsive on mobile networks.

## Authorization-Aware UI Rules

The frontend should reflect authorization, but never replace server enforcement.

Rules:

- derive current trip role from API payloads such as `TripDetailResponse.currentUserRole`
- hide or disable owner/editor actions when the current role cannot perform them
- keep view-only screens readable for viewers without presenting misleading editable affordances
- show explicit messaging when an action is unavailable because of role, not because of a generic error
- treat `403 FORBIDDEN` as a normal UI case that can occur even if the action looked available earlier
- after a `403`, refetch relevant trip detail and membership data because role state may have changed
- do not encode authorization logic in scattered constants across components; centralize role capability helpers per feature

Suggested role affordance model:

- `owner`
  - full trip management
  - sharing changes
  - destructive trip actions
- `editor`
  - edit most trip content
  - no member-role management
  - no trip deletion or owner-only archival actions
- `viewer`
  - read-only access to trip content
  - no create, edit, delete, or sharing actions

## State Management Principles

- Treat server state and client state separately.
- Use TanStack Query for server state, not a custom global store.
- Keep local UI state close to the component or page that owns it.
- Use URL search params for user-meaningful filter and sort state where shareable navigation matters.
- Avoid introducing Redux, Zustand, or another app-wide store unless a real cross-route client-only state problem appears.
- Prefer derived state from query results over duplicated normalized client caches.
- Reset feature-local transient state when route context changes from one trip to another.
- Keep form draft state inside forms, not in global stores.

In practice:

- server data: React Query
- auth/session: Clerk plus thin current-user query wrappers
- route state: React Router params and search params
- ephemeral UI state: `useState`, `useReducer`, or feature-local context only where needed

## Accessibility Baseline

The MVP frontend should meet a practical accessibility baseline from the start.

Required rules:

- all interactive controls must be keyboard reachable
- all form inputs must have labels, descriptions, and error associations
- dialogs, menus, and popovers should use accessible primitives rather than custom ad hoc implementations
- color must not be the only indicator of role, error, completion, or status
- loading and mutation feedback should be announced where appropriate through semantic markup or ARIA live regions
- page titles and heading hierarchy should be consistent by route
- destructive confirmations must be explicit and understandable with screen readers
- link text should describe destination or action, not use vague labels like `Click here`

Use Radix primitives and shared `packages/ui` wrappers to reduce accessibility regressions in common interactions.

## Testing Strategy for the Frontend

The frontend test strategy should match the MVP's risk profile and existing monorepo defaults.

### Unit and component tests

Use:

- `Vitest`
- `React Testing Library`

Cover:

- shared UI primitives used by the app
- feature components with meaningful conditional rendering
- forms with validation and submit-state behavior
- role-aware action visibility
- empty, loading, and inline error states

### Integration-style screen tests

Use component-level integration tests around routed pages with mocked API responses.

Cover:

- trips list loading and empty states
- trip overview summary rendering
- checklist item completion flow
- reservation form validation and successful submit
- sharing screen role restrictions
- budget summary refresh after mutation

### End-to-end tests

Use:

- `Playwright`

Limit E2E coverage to critical user journeys:

- sign in and bootstrap session
- create trip
- edit trip metadata
- add itinerary item
- create reservation
- create checklist and complete item
- create budget entry
- share trip with an existing user

### Test data and mocking rules

- mock Clerk session state in unit and integration tests
- mock network responses at the API boundary, not internal hook details
- keep fixtures small and trip-scoped so authorization expectations remain obvious
- add regression tests for any role-sensitive or mobile-layout-sensitive bug

## Final Architecture Guidance

- Keep the web app organized by feature and route, not by technical layer alone.
- Keep transport boundaries explicit through a thin API client and React Query hooks.
- Keep reusable UI accessible and presentation-focused in `packages/ui`.
- Keep the app mobile-first, trip-centric, and role-aware.
- Keep client-side state minimal and intentional.
- Keep authorization as a UI affordance concern on the frontend and an enforcement concern on the API.

This architecture should be treated as the baseline for `apps/web` implementation and for downstream tasks involving routing, page composition, frontend libraries, or test coverage.
