You are working in a greenfield monorepo for a Travel Planner MVP.

Product context:
- This is a collaborative trip planning web app for families and small groups.
- MVP includes trips, travelers, itinerary, reservations, checklists, notes/links, budget tracking, sharing, and reminders.
- Keep the product simple, operational, and mobile-friendly.
- Do not expand scope into bookings marketplace, advanced AI itinerary generation, native mobile, or enterprise features.

Locked technical direction:
- Monorepo: pnpm + Turborepo
- Frontend: React + TypeScript + Vite
- Backend: NestJS + TypeScript
- Database: PostgreSQL with Prisma
- Frontend hosting: Azure Static Web Apps
- Backend hosting: Azure Container Apps
- Async messaging: Azure Service Bus
- Secrets: Azure Key Vault
- Observability: OpenTelemetry + Azure Monitor / Application Insights
- Infra as code: Bicep

Execution rules:
1. Treat the locked stack as a constraint, not a suggestion.
2. Prefer simple, boring, production-ready choices.
3. Keep the MVP small and avoid speculative abstractions.
4. Design for autonomous agent execution: outputs must be explicit, actionable, and decomposable.
5. When writing docs, be concrete about boundaries, entities, contracts, dependencies, and acceptance criteria.
6. Do not leave placeholders like “TBD” unless absolutely necessary. If uncertain, make a reasonable decision and state it.
7. Write files exactly where requested.
8. If updating an existing file, preserve useful content and improve it instead of rewriting blindly.