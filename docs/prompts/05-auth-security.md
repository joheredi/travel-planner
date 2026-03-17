<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/05-auth-security.md`.

Inputs:
- `docs/architecture/00-stack-decision.md`
- `docs/architecture/01-system-overview.md`
- `docs/architecture/02-domain-model.md`
- `docs/architecture/03-api-design.md`

Goal:
Design authentication, authorization, and baseline security for the MVP.

Requirements:
1. Recommend a concrete auth approach compatible with the locked stack and Azure deployment
2. Define:
   - user identity model
   - session/token model
   - frontend auth flow
   - backend auth validation flow
3. Define authorization rules:
   - trip owner
   - editor
   - viewer
4. Define invitation and trip-sharing security model
5. Define secret handling strategy using Azure Key Vault
6. Define basic security controls:
   - input validation
   - rate limiting
   - CORS
   - CSRF assumptions
   - secure headers
   - audit logging for critical actions
7. Define privacy and data exposure rules for shared trips
8. Call out anything intentionally deferred from MVP

Also include:
- a threat checklist for the MVP
- implementation guardrails for backend and frontend agents