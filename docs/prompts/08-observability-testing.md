<REUSABLE INSTRUCTION BLOCK>

Task:
Create `docs/architecture/08-observability-testing.md`.

Inputs:
- all files under `docs/architecture/`

Goal:
Define the quality strategy for the MVP.

The document must include:
1. Test pyramid for this project
   - unit
   - integration
   - API contract tests
   - frontend component/page tests
   - end-to-end tests
2. What each layer must cover
3. Test data strategy
4. Local development test workflow
5. OpenTelemetry instrumentation expectations
6. Logging standards
7. Metrics and traces expectations
8. Key dashboards/alerts to create in Application Insights
9. Release quality gates
10. Definition of done checklist for tasks

Also include:
- minimum required tests per task type
- observability requirements for mutations, async processing, and failures