# Authentication, Authorization, and Security for Travel Planner MVP

## Purpose

This document defines the authentication model, authorization rules, and baseline security controls for the Travel Planner MVP. It builds on:

- `docs/architecture/00-stack-decision.md`
- `docs/architecture/01-system-overview.md`
- `docs/architecture/02-domain-model.md`
- `docs/architecture/03-api-design.md`

The goal is to use a concrete auth approach that fits the locked stack, keeps identity outsourced to Clerk, keeps authorization inside the NestJS API, and establishes a practical security baseline for a small collaborative trip planning product.

## Security Principles

- Keep identity management outsourced to Clerk.
- Keep authorization decisions server-side in the API.
- Treat every trip-scoped API request as untrusted until membership and role checks pass.
- Minimize browser exposure of sensitive data and secrets.
- Keep the MVP secure with boring, explicit controls instead of speculative enterprise features.
- Prefer deny-by-default behavior for both authorization and data exposure.
- Log and trace critical auth and sharing actions for auditability.

## Recommended Authentication Approach

Use Clerk as the sole identity provider for the MVP across the React SPA and NestJS API.

Concrete approach:

- the SPA integrates Clerk's frontend SDK for sign-in, sign-up, password reset, and session lifecycle
- Clerk manages browser session cookies on its own domain and client session state in the browser
- the SPA obtains a short-lived Clerk JWT for API calls
- the SPA sends that token as `Authorization: Bearer <Clerk JWT>` to the NestJS API
- the API validates the token against Clerk configuration and resolves or creates the local application `User`
- the local `User` record and `TripMember` records remain the source of application permissions

This matches the locked stack and the existing API design:

- the frontend remains a Vite SPA
- the API remains a NestJS REST service
- the API does not expose login or logout endpoints
- the API uses bearer-token auth on all SPA-facing endpoints

## User Identity Model

### Identity boundary

Clerk is the source of truth for:

- sign-up and sign-in
- password reset
- email verification flows that Clerk provides
- session issuance and session revocation
- primary authentication credentials

Travel Planner is the source of truth for:

- application user profile fields that are local to the product
- trip memberships
- trip roles
- authorization and sharing behavior

### Application user model

The local `User` entity should remain the application identity record linked to Clerk.

Required fields:

- `id` - internal UUID
- `clerkUserId` - unique external identity key
- `email` - normalized unique email
- `displayName`
- `avatarUrl?`
- `defaultTimezone?`
- `createdAt`
- `updatedAt`

Rules:

- `clerkUserId` is the stable foreign identity key
- `email` is used for display and sharing lookups, but authorization should not depend on email string comparisons after bootstrap
- local authorization joins should use internal `User.id`
- a user may exist in Clerk before they become a collaborator on any trip

### Identity lifecycle

1. User signs in or signs up with Clerk.
2. SPA obtains a valid Clerk session.
3. SPA calls `POST /v1/session/sync`.
4. API verifies the Clerk token and upserts or resolves the local `User`.
5. API returns the local profile plus membership summary.
6. All later trip authorization uses local `User.id` plus `TripMember` records.

This keeps external identity and internal authorization clearly separated.

## Session and Token Model

### Browser session model

The browser session is Clerk-managed. The Travel Planner SPA should not create its own long-lived browser session cookie for API auth.

Browser rules:

- rely on Clerk for sign-in persistence and session state
- do not store backend secrets in browser storage
- do not store custom refresh tokens for the API in local storage
- only request tokens from Clerk when needed for API calls

### API token model

Use short-lived Clerk JWT bearer tokens for all SPA-to-API requests.

Rules:

- tokens are sent in the `Authorization` header
- tokens are validated on every authenticated API request
- tokens must include the expected issuer and audience configuration for the API
- the API should reject expired, malformed, or incorrectly scoped tokens with `401 UNAUTHENTICATED`
- the API should not accept unsigned or self-issued tokens under any circumstances

### Token validation inputs

The API should validate, at minimum:

- signature
- issuer
- audience
- expiration
- subject / Clerk user identifier

Prefer using Clerk's supported backend verification path rather than hand-rolled JWT validation logic.

## Frontend Authentication Flow

### Sign-in flow

1. User opens the SPA.
2. If not authenticated, the SPA routes the user to Clerk sign-in UI.
3. Clerk completes authentication and establishes a browser session.
4. The SPA renders the authenticated shell.
5. The SPA calls `POST /v1/session/sync` using a Clerk bearer token.
6. The API returns the local user profile and membership summary.
7. The SPA routes the user to `/trips`.

### Ongoing authenticated requests

- the SPA obtains a fresh token from Clerk when making API requests
- the API client attaches the token to every authenticated call
- on `401`, the frontend should treat the session as invalid or expired and route back through the auth boundary
- on `403`, the frontend should show a permission-aware message and refetch relevant trip context if needed

### Frontend implementation rules

- use Clerk's React provider at the app root
- protect authenticated routes with a route guard tied to Clerk session state
- call `POST /v1/session/sync` once after a fresh authenticated bootstrap, not before every request
- keep current-user profile data in server state, not in a custom global auth store
- never treat hidden buttons as security; the backend remains authoritative

## Backend Authentication Validation Flow

For every authenticated request:

1. Read the bearer token from the `Authorization` header.
2. Reject missing or malformed auth headers with `401 UNAUTHENTICATED`.
3. Verify the Clerk JWT using approved backend validation helpers or Clerk's JWKS-backed validation flow.
4. Extract the Clerk subject and relevant claims.
5. Resolve the local `User` by `clerkUserId`.
6. For `POST /v1/session/sync`, create or update the local `User` as needed.
7. For other routes, require a resolvable local `User` and reject missing mappings with an explicit auth/domain error.
8. Pass the authenticated local user context into downstream services.
9. Perform resource-specific authorization checks before any domain mutation or detail read.

Backend rules:

- do not trust role or trip membership claims from the browser
- do not rely on frontend-provided user IDs for authorization
- do not let controllers skip auth guards for trip-scoped routes
- keep authorization checks close to service methods or dedicated guards/policies so they are consistent across endpoints

## Authorization Rules

Authorization is domain-driven and based on `TripMember.role`, not on Clerk claims.

### Core role model

- `owner`
- `editor`
- `viewer`

### Owner

Owners can:

- view all trip-scoped content
- create, update, and delete trip content
- manage travelers, itinerary items, reservations, checklists, notes, links, budget entries, and reminders
- add trip members
- change member roles subject to invariants
- revoke member access
- archive or delete the trip

Owner restrictions:

- cannot create a second owner through the MVP sharing flow
- cannot remove or demote the last owner

### Editor

Editors can:

- view all trip-scoped content
- create, update, and delete ordinary trip content
- manage travelers, itinerary items, reservations, checklists, notes, links, budget entries, and reminders

Editors cannot:

- add, remove, or re-role trip members
- archive a trip if that action is owner-only
- delete the trip

### Viewer

Viewers can:

- read trip-scoped content they are authorized to see
- view trip overview, itinerary, reservations, checklists, notes, links, budget, and sharing detail

Viewers cannot:

- create, edit, or delete trip content
- manage reminders
- manage membership
- archive or delete trips

### Authorization evaluation rules

- evaluate membership against the specific `tripId`
- return `404` when the resource is not in the caller's accessible scope, consistent with the API design
- return `403` when the caller is authenticated but lacks the required action permission for an otherwise visible resource
- re-check authorization inside the service layer before executing mutations
- validate all cross-resource references belong to the same trip

## Invitation and Trip-Sharing Security Model

The MVP uses a deliberately narrow sharing model.

### Sharing model

- trip sharing is owner-only
- collaborators must already have a Clerk-backed Travel Planner account
- the owner shares by normalized email address
- the API resolves the local `User` by normalized email
- if found, the API creates or updates the `TripMember`
- there is no pending invitation entity in the MVP
- there are no public share links in the MVP

### Security rules for sharing

- normalize email before lookup and uniqueness checks
- keep `POST /v1/trips/:tripId/members` idempotent by effective `tripId + email`
- reject attempts to add a second owner
- reject removal or demotion of the last owner
- log membership changes as security-relevant audit events
- return a controlled `404 USER_NOT_FOUND` if the target email has no local account
- do not expose whether an email exists outside the authenticated owner sharing workflow

### Data exposure during sharing

Members list responses should include only data needed for collaboration UI:

- local member ID
- display name
- email
- role
- invitation status
- timestamps needed for UI or audit display

Do not expose:

- Clerk session identifiers
- raw JWT claims beyond what the app needs
- backend-only secrets or internal auth metadata

## Secret Handling Strategy with Azure Key Vault

Azure Key Vault is the only approved production secret source.

### What belongs in Key Vault

- Clerk secret key
- Clerk JWT verification configuration if needed as a secret
- database credentials
- Azure Service Bus credentials or connection settings if not using managed identity
- Azure Communication Services email credentials
- any API or worker signing secrets introduced later

### What does not belong in the frontend

- Clerk secret key
- database credentials
- queue credentials
- email provider credentials
- any token verification secret or private key

Browser-visible configuration must be limited to intentionally public values such as:

- Clerk publishable key
- API base URL
- non-sensitive telemetry or environment labels

### Runtime rules

- inject server secrets into `apps/api` and `apps/worker` through deployment configuration backed by Key Vault
- validate required env vars at startup
- commit `.env.example` files only with placeholders and non-secret names
- never copy production secrets into local checked-in files
- prefer Azure managed identity to reduce raw secret distribution where the platform integration supports it

## Basic Security Controls

### Input validation

- use NestJS global `ValidationPipe` with `transform`, `whitelist`, and `forbidNonWhitelisted`
- validate DTOs explicitly with `class-validator`
- parse UUID path params with `ParseUUIDPipe`
- validate enums and date ranges explicitly
- enforce business invariants in services, not only at DTO level
- normalize emails and similar identifiers before persistence and comparison
- reject cross-trip resource references explicitly

### Rate limiting

Baseline rate limiting should exist at two layers:

1. Edge or platform layer
   - apply coarse protection against bursts, abuse, and obvious unauthenticated flooding at the ingress boundary

2. API layer
   - apply route-sensitive throttling in NestJS, especially for:
     - `POST /v1/session/sync`
     - `GET/PATCH /v1/me`
     - sharing endpoints
     - reminder creation endpoints

Rules:

- authenticated limits may be keyed by user ID where possible
- unauthenticated limits should fall back to IP or platform identity
- do not silently drop throttled requests; return explicit rate-limit responses

### CORS

Use a strict allowlist for CORS in the API.

Rules:

- allow only the known SPA origin for each environment
- allow credentials only if a specific integration requires them
- allow only required methods and headers, including `Authorization` and `Content-Type`
- do not use `*` origins in production
- keep local development origins explicit and environment-scoped

### CSRF assumptions

The public SPA-to-API model uses bearer tokens in the `Authorization` header, not a same-site API session cookie. Because of that:

- classic browser CSRF is not the primary risk for API routes
- CSRF tokens are not required for normal API requests in the MVP under this model
- if the architecture later introduces cookie-authenticated API endpoints, that assumption must be revisited immediately

Important caveat:

- Clerk's own browser-based auth flows remain governed by Clerk's controls and should not be reimplemented by the product

### Secure headers

At minimum, the frontend hosting layer and API should enforce:

- `Strict-Transport-Security`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy` with a restrictive setting
- `Content-Security-Policy` appropriate for the SPA, Clerk assets, and approved telemetry endpoints
- `X-Frame-Options` or equivalent CSP frame control to prevent clickjacking where embedding is not required

Rules:

- serve only over HTTPS in non-local environments
- keep CSP as strict as practical and document any required third-party origins
- do not allow arbitrary inline scripts unless there is a concrete, reviewed need

### Audit logging for critical actions

The API should emit audit-friendly logs and traces for:

- session sync user creation or email-conflict events
- trip creation and deletion
- trip archival
- membership add, role change, and revoke actions
- reminder creation, update, and cancel actions
- repeated authorization failures

Audit log rules:

- include request ID and authenticated local user ID
- include trip ID and target resource ID when relevant
- do not log raw bearer tokens, passwords, or full secret material
- avoid logging sensitive freeform note bodies or other unnecessary PII

## Privacy and Data Exposure Rules for Shared Trips

Shared trips are collaborative, but data exposure should still be intentionally limited.

### Baseline visibility

Active trip members may view:

- trip metadata
- itinerary items
- reservations
- checklists
- notes and links
- budget entries
- trip member list
- reminder and notification records that are part of the trip context

### Exposure rules

- expose only the fields required by the UI for each list and detail response
- keep list payloads lightweight and mobile-oriented
- do not expose internal database fields, Clerk internals, or operational secrets
- do not expose non-member trip data to any authenticated user outside that trip
- keep traveler data trip-scoped and visible only to trip members
- keep member list data limited to collaboration-relevant identity fields
- avoid returning hidden or archived resources unless the endpoint explicitly supports that state

### Logging and telemetry privacy rules

- do not put note bodies, reservation confirmation codes, or budget notes into high-cardinality telemetry by default
- log IDs, codes, counts, and decision outcomes instead of full content payloads where possible
- include correlation IDs so security-relevant actions can be traced without over-logging personal content

## Threat Checklist for the MVP

The following threats must be checked during implementation and review:

- token forgery or invalid token acceptance
- missing trip membership checks on trip-scoped endpoints
- privilege escalation through editor or viewer paths
- insecure direct object reference across trips
- sharing abuse through unnormalized email lookups
- accidental creation of multiple owners or removal of the last owner
- leaking secrets into frontend bundles or logs
- CORS misconfiguration that allows untrusted origins
- overly broad CSP exceptions or unsafe inline script usage
- brute-force or flood behavior against auth-adjacent endpoints
- injection through malformed input, URLs, or freeform content
- overexposure of member, traveler, or reminder data in list responses
- replay or duplicate effects on non-idempotent mutations
- sensitive content appearing in logs, traces, or client-side error reports

## Intentionally Deferred from MVP

The MVP explicitly defers:

- custom credential storage or self-hosted identity
- SSO, SAML, or enterprise directory integration
- public invitation links
- invitation acceptance flows with pending invites
- multi-factor requirements controlled by Travel Planner itself beyond what Clerk provides
- field-level encryption for ordinary application data
- row-level security implemented directly in PostgreSQL as the primary authorization model
- in-app security center features
- advanced anomaly detection or risk scoring
- device management and user session management UI beyond Clerk defaults

These items may become valid later, but they should not complicate the MVP baseline now.

## Implementation Guardrails for Backend and Frontend Agents

### Backend guardrails

- always validate Clerk JWTs before reading protected route inputs as authenticated user context
- always resolve the local `User` and authorize against `TripMember`
- never trust role, membership, or ownership values supplied by the client
- enforce same-trip invariants on every related resource reference
- centralize role checks so multiple controllers cannot drift
- return explicit auth and validation errors; do not silently ignore unauthorized fields
- never log raw auth headers or secret values
- use Key Vault-backed configuration for production secrets
- add tests for owner, editor, viewer, and non-member cases on every new trip-scoped endpoint

### Frontend guardrails

- use Clerk only for authentication UI and session state, not as the source of application permissions
- never hardcode trust in frontend role gating; treat it as affordance only
- always attach bearer tokens through the shared API client rather than ad hoc fetch calls
- handle `401` and `403` distinctly in UI behavior
- do not persist backend secrets, JWTs, or sensitive content in local storage unnecessarily
- do not expose hidden admin or destructive actions based solely on route knowledge
- prefer least-data rendering and avoid requesting detail endpoints until the UI needs them
- show permission-aware messaging without leaking protected data

## Final Design Guidance

- Keep Clerk responsible for identity.
- Keep the NestJS API responsible for token validation and authorization.
- Keep trip access modeled through local `TripMember` records.
- Keep sharing owner-controlled and limited to existing users in the MVP.
- Keep secrets in Azure Key Vault and out of the browser.
- Keep security controls explicit: validation, rate limiting, strict CORS, secure headers, and audit logging.
- Keep privacy rules conservative for shared-trip data.

This document should be treated as the baseline for implementation tasks that touch auth, trip sharing, request security, secrets, or privacy-sensitive data handling.
