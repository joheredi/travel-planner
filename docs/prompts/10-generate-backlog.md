<REUSABLE INSTRUCTION BLOCK>

Task:
Generate implementation tasks under `docs/backlog/tasks/*.md` and create `docs/backlog/index.md`.

Inputs:
- `docs/prd/travel-planner-prd.md`
- all files under `docs/architecture/`
- all files under `docs/backlog/milestones/`

Goal:
Create an execution-ready backlog for autonomous AI agents.

Requirements:
1. Create tasks that are small, bounded, and independently executable where possible
2. Prefer vertical slices, but include foundational tasks when necessary
3. Every task must belong to one milestone
4. Every task must have an ID like `B001`, `B002`, etc.
5. Every task must have its own file in `docs/backlog/tasks/`
6. `docs/backlog/index.md` must be an index only and must not duplicate full task content

Each task file must include these sections exactly:
- Title
- ID
- Milestone
- Objective
- Background
- Scope
- Out of Scope
- Dependencies
- Target Files / Areas
- Implementation Notes
- Acceptance Criteria
- Tests Required
- Observability Requirements
- Documentation Updates
- Agent Notes

Task writing rules:
- Tasks must be scoped so a single AI agent can complete them in one focused work stream
- Each task must reference the architecture docs it depends on
- Each task must be implementation-specific, not vague
- Include expected file paths or code areas whenever possible
- Include explicit acceptance criteria that can be verified
- Include required test coverage
- Include required telemetry/logging changes if relevant
- Mark blocker dependencies clearly
- Avoid tasks that mix too many layers unless they are intentionally vertical slices

For `docs/backlog/index.md`, include:
- short project summary
- milestone list
- a compact table or bullet index of all tasks with:
  - ID
  - title
  - milestone
  - dependency summary
  - status placeholder `[ ]`

Important:
- generate enough tasks to implement the MVP, but keep tasks reasonably granular
- do not generate “epic” placeholders
- do not generate “investigate X” tasks unless truly necessary