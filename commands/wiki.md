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

## Subcommand: `update <section>`

Analyze the codebase to generate content for the specified section. Each section has a different analysis strategy:

### `architecture`
1. Read `package.json` (or `.csproj`, `pyproject.toml`) for tech stack
2. Scan directory structure: `ls -R src/ app/ lib/ 2>/dev/null | head -50`
3. Look for patterns: `grep -r "export default" src/ --include="*.ts" -l | head -20`
4. Identify layers, patterns, key abstractions
5. Write findings to `docs/wiki/architecture.md`

### `api-reference`
1. Find API route files: `find . -path "*/api/*" -name "*.ts" -o -name "*.py" -o -name "*.cs" 2>/dev/null`
2. For Next.js: scan `app/api/` for route handlers
3. For .NET: scan for `[ApiController]` or `[HttpGet]` attributes
4. Extract endpoints, methods, parameters
5. Write to `docs/wiki/api-reference.md`

### `data-model`
1. Look for schema files: Prisma schema, EF Core entities, SQLAlchemy models
2. Scan for model/entity definitions
3. Document entities, fields, relationships
4. Write to `docs/wiki/data-model.md`

### `services`
1. Find service files: `grep -r "Service" src/ --include="*.ts" -l 2>/dev/null`
2. Look for background jobs, cron tasks, external integrations
3. Document each service's purpose and dependencies
4. Write to `docs/wiki/services.md`

### `testing`
1. Find test files: `find . -name "*.test.*" -o -name "*.spec.*" 2>/dev/null`
2. Read test config: `vitest.config.*`, `jest.config.*`, `playwright.config.*`
3. Check `package.json` for test scripts
4. Run test count: `npm test -- --reporter=verbose 2>&1 | tail -5`
5. Document strategy, structure, how to run
6. Write to `docs/wiki/testing.md`

### `deployment`
1. Look for: `Dockerfile`, `docker-compose.*`, `azure-pipelines.yml`, `vercel.json`
2. Read deployment config files
3. Check `.env.claude` for deploy target and URLs
4. Document environments, pipeline stages, manual steps
5. Write to `docs/wiki/deployment.md`

### `configuration`
1. Find env files: `.env.example`, `.env.claude.example`
2. Find config files: `appsettings.json`, `next.config.*`
3. Scan for `process.env.` or `Environment.GetEnvironmentVariable` usage
4. Document all environment variables with descriptions
5. Write to `docs/wiki/configuration.md`

### `conventions`
1. Read `.eslintrc`, `.prettierrc`, `tsconfig.json`, `CLAUDE.md`
2. Check for naming patterns in existing code
3. Read git log for commit message format: `git log --oneline -20`
4. Document naming, git, coding standards
5. Write to `docs/wiki/conventions.md`

### After updating any section:
1. Update the `status` field in frontmatter to `draft`
2. Update the `last_updated` field to today's date
3. Report what was written

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
