# Hermes Specs

This directory contains the functional, technical, and architectural specifications for **Hermes API Platform**.

The purpose of this directory is to keep project planning, implementation scope, acceptance criteria, and architectural decisions close to the source code. These documents are intended to guide both human contributors and AI-assisted development agents.

## Directory structure

```text
specs/
├── README.md
├── 000-project-vision.md
├── 001-project-bootstrap/
│   ├── US-001-project-bootstrap.md
│   ├── TASK-001-parent-maven-project.md
│   ├── TASK-002-maven-wrapper.md
│   ├── TASK-003-hermes-common-module.md
│   ├── TASK-004-hermes-gateway-module.md
│   └── TASK-005-actuator-health-endpoint.md
│
├── 002-gateway-mvp/
│   ├── US-002-gateway-mvp.md
│   ├── TASK-001-static-route-configuration.md
│   ├── TASK-002-programmatic-route-configuration.md
│   ├── TASK-003-correlation-id-filter.md
│   └── TASK-004-request-logging-filter.md
│
├── 003-resilience-rate-limiting/
│   ├── US-003-resilience-and-rate-limiting.md
│   ├── TASK-001-docker-compose-redis.md
│   ├── TASK-002-token-bucket-rate-limiting.md
│   ├── TASK-003-circuit-breaker-fallback.md
│   └── TASK-004-integration-tests.md
│
├── 004-admin-api/
│   ├── US-004-admin-api.md
│   ├── TASK-001-hermes-admin-module.md
│   ├── TASK-002-route-crud.md
│   ├── TASK-003-tenant-crud.md
│   └── TASK-004-api-key-management.md
│
└── adr/
    ├── ADR-001-use-spring-cloud-gateway.md
    ├── ADR-002-use-redis-for-rate-limiting.md
    └── ADR-003-keep-main-as-production-branch.md
```

## Document types

| Prefix | Type | Purpose |
|---|---|---|
| `US-*` | User Story / Technical Story | Describes the value, scope, and acceptance criteria of a feature or technical milestone. |
| `TASK-*` | Implementation Task | Describes a concrete technical action required to complete a story. |
| `ADR-*` | Architecture Decision Record | Documents an architectural decision, its context, options, and consequences. |

## Naming convention

Use this format for folders:

```text
specs/{milestone-number}-{milestone-name}/
```

Examples:

```text
specs/001-project-bootstrap/
specs/002-gateway-mvp/
specs/003-resilience-rate-limiting/
```

Use this format for documents:

```text
US-{number}-{short-description}.md
TASK-{number}-{short-description}.md
ADR-{number}-{short-description}.md
```

Examples:

```text
US-001-project-bootstrap.md
TASK-001-parent-maven-project.md
ADR-001-use-spring-cloud-gateway.md
```

## Status values

Each story, task, or ADR should use one of the following statuses:

| Status | Meaning |
|---|---|
| `Planned` | Documented but not started. |
| `In Progress` | Currently being worked on. |
| `Done` | Completed and validated. |
| `Deferred` | Postponed for a future milestone. |
| `Blocked` | Cannot continue until a dependency or decision is resolved. |

## AI agent instructions

AI-assisted work on this repository must follow these rules:

1. Read the relevant `US-*` document before implementing any `TASK-*`.
2. Do not implement work marked as `Out of scope`.
3. Follow the acceptance criteria exactly.
4. Prefer small, incremental changes.
5. Keep `main` stable.
6. Do not introduce infrastructure, security, database, or cloud dependencies unless the current task explicitly requires them.
7. Update the corresponding spec document when the implementation changes the original scope.
8. Do not mark a task as `Done` unless the validation section has been completed.
9. When creating code, follow the project package convention: `dev.eithel.hermes`.
10. When creating commits, use Conventional Commits.

## User Story template

Use this template for every `US-*` document.

```markdown
# US-XXX — Story title

## Status

Planned

## Summary

As a [role],
I want [capability or goal],
so that [expected value or outcome].

## Business / Engineering value

Explain why this story matters.

Focus on the value it brings to the project architecture, maintainability, developer experience, production readiness, or user-facing behavior.

## Scope

This story includes:

- Item 1.
- Item 2.
- Item 3.

## Out of scope

This story does not include:

- Item 1.
- Item 2.
- Item 3.

## Acceptance criteria

- [ ] Criterion 1.
- [ ] Criterion 2.
- [ ] Criterion 3.

## Technical notes

Add relevant technical guidance, constraints, package names, module names, configuration expectations, or implementation considerations.

## Related tasks

- `TASK-001-example-task.md`
- `TASK-002-example-task.md`

## Dependencies

List dependencies on previous stories, tasks, modules, infrastructure, or decisions.

If there are no dependencies, write:

- None.

## Validation

Describe how this story should be validated once all related tasks are completed.

Examples:

```bash
./mvnw clean verify
```

```bash
curl http://localhost:8080/actuator/health
```

## Notes

Add any additional context, assumptions, or open questions.
```

## Task template

Use this template for every `TASK-*` document.

```markdown
# TASK-XXX — Task title

## Status

Planned

## Parent story

`US-XXX-story-name.md`

## Summary

Describe the concrete technical task to be completed.

## Technical scope

This task includes:

- Item 1.
- Item 2.
- Item 3.

## Out of scope

This task does not include:

- Item 1.
- Item 2.
- Item 3.

## Implementation notes

Describe the expected implementation approach.

Include relevant:

- Module name.
- Package name.
- Class names.
- Configuration files.
- Dependencies.
- Constraints.

## Expected files

List files or directories expected to be created or modified.

```text
example-module/
└── src/main/java/dev/eithel/hermes/example/
    └── ExampleClass.java
```

## Acceptance criteria

- [ ] Criterion 1.
- [ ] Criterion 2.
- [ ] Criterion 3.

## Validation

Commands, endpoint checks, tests, or manual verification steps required to consider this task complete.

Examples:

```powershell
.\mvnw.cmd clean validate
```

```powershell
.\mvnw.cmd -pl hermes-gateway spring-boot:run
```

```bash
curl http://localhost:8080/actuator/health
```

## Commit suggestion

Use a Conventional Commit message.

Example:

```text
chore: add parent Maven project
```

## Notes

Add any additional context, assumptions, risks, or follow-up tasks.
```

## ADR template

Use this template for every `ADR-*` document.

```markdown
# ADR-XXX — Decision title

## Status

Proposed

## Context

Describe the problem, constraint, or architectural concern that requires a decision.

## Decision

Describe the chosen decision clearly.

## Options considered

### Option 1 — Name

Pros:

- Advantage 1.
- Advantage 2.

Cons:

- Disadvantage 1.
- Disadvantage 2.

### Option 2 — Name

Pros:

- Advantage 1.
- Advantage 2.

Cons:

- Disadvantage 1.
- Disadvantage 2.

## Consequences

Describe the impact of the decision.

Include:

- Benefits.
- Trade-offs.
- Operational implications.
- Future constraints.

## Related documents

- `US-XXX-example.md`
- `TASK-XXX-example.md`
```

## Recommended first milestone

Start with the project bootstrap milestone:

```text
specs/
└── 001-project-bootstrap/
    ├── US-001-project-bootstrap.md
    ├── TASK-001-parent-maven-project.md
    ├── TASK-002-maven-wrapper.md
    ├── TASK-003-hermes-common-module.md
    ├── TASK-004-hermes-gateway-module.md
    └── TASK-005-actuator-health-endpoint.md
```

This keeps the first implementation focused on the foundation of the project before adding gateway routing, filters, Redis, security, PostgreSQL, Admin API, CI/CD, or infrastructure.

## Recommended workflow

1. Create or update the relevant spec.
2. Create a branch for the task.
3. Implement the smallest possible change.
4. Validate using the task's validation section.
5. Update the task status.
6. Open a Pull Request.
7. Merge only when the acceptance criteria are met.

Example branch names:

```text
docs/project-specs
chore/project-bootstrap
feature/gateway-base
feature/correlation-id-filter
```

Example commit messages:

```text
docs: add initial project specifications
chore: bootstrap Maven multi-module project
feat: add Spring Cloud Gateway base application
```
