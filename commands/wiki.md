---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent]
---

# /wiki — Project Wiki Management

> **Expert Voice:** Technical Writer — keeps documentation accurate, structured, and in sync with the codebase. Writes for developers, not managers.

You manage the project wiki at `docs/wiki/`. The wiki is the single source of truth for developer knowledge. Your job is to keep it accurate and useful.

**Usage:**
- `/wiki` or `/wiki status` — Show wiki status (which sections are populated vs template)
- `/wiki update <section>` — Update a specific section by analyzing the codebase
- `/wiki auto` — Auto-populate all template sections from codebase analysis
- `/wiki review` — Check all sections for accuracy against current codebase

The `$ARGUMENTS` parameter contains the subcommand.

## Step 1: Load Configuration

```bash
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
```

## Step 2: Parse Subcommand

Parse `$ARGUMENTS` to determine the action:

| Input | Action |
|-------|--------|
| (empty) or `status` | Show wiki status |
| `update architecture` | Update `docs/wiki/architecture.md` |
| `update api-reference` | Update `docs/wiki/api-reference.md` |
| `update data-model` | Update `docs/wiki/data-model.md` |
| `update services` | Update `docs/wiki/services.md` |
| `update testing` | Update `docs/wiki/testing.md` |
| `update deployment` | Update `docs/wiki/deployment.md` |
| `update configuration` | Update `docs/wiki/configuration.md` |
| `update conventions` | Update `docs/wiki/conventions.md` |
| `auto` | Update ALL sections that have status `template` |
| `review` | Check all sections for accuracy |

## Subcommand: `status`

Read each file in `docs/wiki/` and check the `status` field in the frontmatter:

```
/wiki — Status Report
======================
Project: $AZURE_DEVOPS_PROJECT

  [reviewed]  architecture.md      (last updated: 2026-03-19)
  [draft]     api-reference.md     (last updated: 2026-03-18)
  [template]  data-model.md        — needs population
  [template]  services.md          — needs population
  [draft]     testing.md           (last updated: 2026-03-19)
  [template]  deployment.md        — needs population
  [template]  configuration.md     — needs population
  [reviewed]  conventions.md       (last updated: 2026-03-17)

Coverage: 4/8 populated (50%)

To populate: /wiki auto
To update one: /wiki update <section-name>
```

## Quality Standard

**Every wiki section MUST include:**
- **Mermaid diagrams** — ER diagrams, sequence diagrams, flowcharts, architecture diagrams as appropriate
- **Detailed tables** — not just lists, but structured tables with all relevant columns
- **Real code examples** — actual snippets from the codebase, not generic placeholders
- **Relationship mapping** — how components connect, data flows, dependency chains
- **Conventions documentation** — patterns the project follows, with examples

**Use the wiki templates in `${CLAUDE_PLUGIN_ROOT:-.}/templates/wiki/` as the structural guide.** Each template shows the expected depth, diagram types, and sections. Replace placeholder content with real data discovered from analysis.

**Think like a Developer Integration Guide:** A new developer joining the team should be able to read the wiki and understand the entire system — architecture, data flow, API contracts, testing patterns, and deployment pipeline — without reading a single line of source code.

## Subcommand: `update <section>`

Analyze the codebase to generate **comprehensive, analytical content** for the specified section. Each section requires deep codebase analysis and must include Mermaid diagrams.

### `architecture`
1. Read `package.json` (or `.csproj`, `pyproject.toml`) for tech stack with exact versions
2. Scan full directory structure — map every layer and its responsibility
3. Read key files to understand patterns: components, hooks, services, middleware, routes
4. Identify all layers, patterns, abstractions, and how they connect
5. **Generate Mermaid diagrams:**
   - System architecture diagram (external actors → application → data)
   - Application layer diagram (presentation → business → data access)
   - Component hierarchy tree
   - Data flow sequence diagram (user action → API → database → response)
   - Security flow diagram (request → auth → authorization → handler)
6. Build technology stack table with exact versions from package.json/config
7. Document key patterns with file locations and examples
8. Document external dependencies and integration points
9. Write to `docs/wiki/architecture.md`

### `api-reference`
1. Find ALL API route files across the project
2. **Read each route handler** — extract method, path, request body, response shape, auth requirement
3. Group endpoints by domain/resource
4. **Generate Mermaid diagrams:**
   - Authentication flow sequence diagram
   - CRUD operation sequence diagram for each major resource
5. Document request/response formats with real JSON examples from the code
6. Build error codes table from actual error handling in the codebase
7. Document pagination, filtering, and sorting patterns
8. Document rate limiting if present
9. Write to `docs/wiki/api-reference.md`

### `data-model`
1. Find schema files: Prisma schema, EF Core entities, SQLAlchemy models, Drizzle schemas
2. **Read every model/entity definition** — extract all fields, types, constraints, defaults
3. Map ALL relationships between entities (1:1, 1:N, N:N)
4. **Generate Mermaid diagrams:**
   - Full ER diagram with all entities, fields (PK/FK/UK marked), and relationship cardinality
   - Data flow diagram (write path and read path)
5. Build detailed entity tables — one subsection per entity with fields, types, constraints, indexes
6. Build relationship matrix table (From → To, type, FK column, cascade behavior)
7. Document migration history and commands
8. Document seed data and how to use it
9. Document database conventions (naming, indexing, soft delete)
10. Write to `docs/wiki/data-model.md`

### `services`
1. Find ALL service files, background workers, scheduled jobs, and integrations
2. **Read each service** — understand purpose, dependencies, trigger mechanism, error handling
3. Map service-to-service and service-to-external dependencies
4. **Generate Mermaid diagrams:**
   - Service architecture diagram (application → services → external)
   - Per-service flow diagram (trigger → validate → process → store → notify)
   - Integration sequence diagram (app → queue → worker → external API → database)
5. Build integration table with protocol, auth, rate limits
6. Document background jobs with schedule, purpose, timeout, monitoring
7. Document health check endpoints
8. Write to `docs/wiki/services.md`

### `testing`
1. Find ALL test files — count by type (unit, integration, E2E)
2. Read test config files for runners, reporters, setup
3. Read actual test files to extract patterns — how tests are structured, mocking approaches
4. **Generate Mermaid diagrams:**
   - Testing pyramid with counts per level
   - Test directory structure tree
   - CI pipeline testing flow
5. Document exact test commands for every scenario (all, single file, watch, coverage, E2E, E2E debug)
6. Extract and document actual test patterns with real code snippets from the project
7. Document mocking patterns with what/how/when table
8. Document coverage targets and current state
9. Write to `docs/wiki/testing.md`

### `deployment`
1. Read ALL deployment config: Dockerfile, docker-compose, azure-pipelines.yml, vercel.json
2. Read `.env.claude` for deploy targets and URLs
3. Analyze pipeline stages from actual YAML/config
4. **Generate Mermaid diagrams:**
   - Full CI/CD pipeline flowchart (push → build → test → deploy → monitor)
   - Deployment sequence diagram (developer → git → CI → environment → monitoring)
   - Infrastructure diagram (DNS → compute → data)
5. Document all environments with branch mapping, URLs, and deploy targets
6. Document pipeline triggers per branch
7. Document rollback procedure with decision matrix (symptom → severity → action)
8. Document environment-specific configuration differences
9. Write to `docs/wiki/deployment.md`

### `configuration`
1. Find ALL env files, config files, and secrets references
2. **Scan the entire codebase** for environment variable usage — build a complete list
3. Categorize variables: required vs optional, per-environment values
4. **Generate Mermaid diagrams:**
   - Configuration flow diagram (sources → loading order → application)
   - Secrets management flow (developer → .env, CI → pipeline vars, production → key vault)
5. Document every variable with type, description, default, and example
6. Document configuration files and their purposes
7. Document secrets management approach
8. Document how to add new configuration
9. Write to `docs/wiki/configuration.md`

### `conventions`
1. Read ALL style/lint config files (.eslintrc, .prettierrc, tsconfig.json, editorconfig)
2. Analyze actual code for naming patterns — sample 20+ files across different layers
3. Read git log for commit message format patterns
4. Read existing CLAUDE.md for documented conventions
5. **Generate Mermaid diagrams:**
   - Component pattern decision flowchart (server vs client component)
   - Gitflow branch diagram with merge strategy
6. Build comprehensive naming convention table (every element type with convention and example)
7. Document code patterns with real examples from the codebase
8. Document error handling pattern with code snippet
9. Document git conventions: branches, commits, PRs
10. Document coding standards: TypeScript, React, CSS
11. Include code review checklist
12. Write to `docs/wiki/conventions.md`

### After updating any section:
1. Update the `status` field in frontmatter to `draft`
2. Update the `last_updated` field to today's date
3. **Verify all Mermaid diagrams render correctly** (valid syntax)
4. Report: section name, diagram count, table count, line count

## Subcommand: `auto`

Run `update` for every section that has `status: template`. Skip sections that are already `draft` or `reviewed`.

Use the Agent tool with `superpowers:dispatching-parallel-agents` to parallelize independent section updates when 3+ sections need updating.

After all updates, print the status report.

## Subcommand: `review`

For each section that is `draft` or `reviewed`:
1. Read the wiki section
2. Verify key claims against the current codebase:
   - Do referenced files still exist?
   - Do referenced functions/classes still exist?
   - Are version numbers current?
3. Report findings:

```
/wiki review — Accuracy Check
===============================
architecture.md:
  [OK] Tech stack matches package.json
  [STALE] References src/utils/helpers.ts — file was renamed to src/lib/helpers.ts

api-reference.md:
  [OK] All endpoints verified
  [MISSING] New endpoint /api/notifications not documented

Actions needed: 2 issues found. Run /wiki update <section> to fix.
```
