## Project Context & Reference Guide

### Configuration
- All lifecycle commands read from `.env.claude` at the repo root
- Tech stack: `{{TECH_STACK}}` | Deploy target: `{{DEPLOY_TARGET}}`
- Azure DevOps: `https://dev.azure.com/{{AZURE_DEVOPS_ORG}}/{{AZURE_DEVOPS_PROJECT}}`

### Project Wiki (docs/wiki/)
The project wiki is the single source of truth for developer knowledge. **Always check relevant sections before implementing.**

| Section | File | When to check |
|---------|------|--------------|
| Architecture | `docs/wiki/architecture.md` | Before adding components, changing data flow, or making structural decisions |
| API Reference | `docs/wiki/api-reference.md` | Before adding/modifying API endpoints |
| Data Model | `docs/wiki/data-model.md` | Before changing database schema, adding entities, or modifying relationships |
| Services | `docs/wiki/services.md` | Before adding background jobs, integrations, or external service calls |
| Testing | `docs/wiki/testing.md` | Before writing tests — follow documented patterns, runners, and conventions |
| Deployment | `docs/wiki/deployment.md` | Before changing CI/CD, Docker, or deployment configuration |
| Configuration | `docs/wiki/configuration.md` | Before adding environment variables or config files |
| Conventions | `docs/wiki/conventions.md` | Before writing any code — follow naming, style, and commit conventions |

### Where to Look Up Context

| Need | Where to look |
|------|--------------|
| **Data models & relationships** | `docs/wiki/data-model.md` — ER diagrams, entity definitions, migration patterns |
| **Architecture & patterns** | `docs/wiki/architecture.md` — layer diagrams, component hierarchy, data flow |
| **API contracts** | `docs/wiki/api-reference.md` — endpoints, request/response schemas, auth |
| **Best practices** | `docs/wiki/conventions.md` — coding standards, naming rules, git workflow |
| **How to test** | `docs/wiki/testing.md` — test structure, mocking patterns, E2E setup |
| **How to deploy** | `docs/wiki/deployment.md` — pipeline stages, environment config, rollback |
| **Project decisions** | `docs/decisions/` — Architecture Decision Records (if they exist) |

### Rules
- When implementing features, ALWAYS check the relevant wiki section first
- When making architectural decisions, update `docs/wiki/architecture.md`
- After adding API endpoints, update `docs/wiki/api-reference.md`
- After changing database schema, update `docs/wiki/data-model.md`
- After modifying tests, update `docs/wiki/testing.md`
- After adding environment variables, update `docs/wiki/configuration.md`
- Keep wiki sections in sync — run `/wiki review` to check accuracy
