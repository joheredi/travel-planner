<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/00-stack-decision.md`.

Inputs:
- `docs/prd/travel-planner-prd.md`

Goal:
Produce a stack decision document that locks the technical foundation for the MVP and removes ambiguity for downstream design and backlog generation.

The document must include:
1. Executive summary of the chosen stack
2. Why this stack fits the PRD
3. Explicitly rejected alternatives and why
   - Next.js full-stack
   - .NET backend
   - MongoDB
   - Azure App Service instead of Azure Container Apps
   - self-hosted auth
   - Kubernetes/AKS
4. Monorepo conventions
   - package manager
   - task runner
   - apps/packages layout
   - env var strategy
   - linting/formatting/testing defaults
5. Deployment topology at a high level
6. Local development standards
7. Definition of done for implementation tasks
8. Non-goals and anti-patterns

Also include a “Locked Decisions” section with concise bullets that downstream prompts can treat as immutable.