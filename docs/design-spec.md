# Portable Development Lifecycle Automation Design

## Overview

A portable package of Claude Code commands and skills that can be dropped into any project and self-configure from a single `.env.claude` file. The system covers the full development lifecycle — from feature ideation through production release — using Azure DevOps as the central platform for boards, repos, and pipelines.

## Goals

- One config file (`.env.claude`) makes the entire command suite project-aware
- Three development tracks: full feature, quick fix, hotfix — each with a clear chain of commands
- Every command orchestrates existing skills rather than reinventing functionality
- Commands speak with the voice of domain experts (BA, QA, SRE, etc.) using standard Claude agent infrastructure
- File-driven state: each command produces artifacts consumed by the next command in the chain
- Support multiple deploy targets (Hetzner, Azure Cloud, Vercel) from a single pipeline template

## Configuration: `.env.claude`

Per-project config file at the repository root. This is the only project-specific file required.

```env
# Azure DevOps
AZURE_DEVOPS_ORG=my-org
AZURE_DEVOPS_PROJECT=my-project
AZURE_DEVOPS_PAT=xxxxx

# Deploy target: hetzner | azure | vercel
DEPLOY_TARGET=hetzner

# Environment URLs
STAGING_URL=https://staging.example.com
PRODUCTION_URL=https://example.com

# Tech stack: nextjs | dotnet | python
TECH_STACK=nextjs

# Branch strategy (default: gitflow)
BRANCH_STRATEGY=gitflow

# Pipeline IDs (populated after /init-project creates them)
STAGING_PIPELINE_ID=
PRODUCTION_PIPELINE_ID=
```

All commands read from this file. Existing commands (`/board`, `/overview`, `/ticket`, `/update-ticket`, `/create-backlog`) are enhanced to read Azure DevOps config from `.env.claude` instead of hardcoded values.

## Development Tracks

### 1. Full Feature Track (new features)

```
/feature → /develop → /staging → /release
```

| Step | Command | Expert Voice | What It Does |
|------|---------|-------------|--------------|
| 1 | `/feature` | Business Analyst + Solution Architect | Brainstorms design, writes plans, creates Azure DevOps work items |
| 2 | `/develop` | Senior Developer | 10-phase TDD implementation, merges to develop |
| 3 | `/staging` | QA Engineer | Promotes develop to staging branch, triggers staging pipeline, runs E2E tests, verifies deployment at staging URL |
| 4 | `/release` | Release Manager | Promotes staging to master, triggers production pipeline, creates git tag, verifies production deployment |

### 2. Quick Fix Track (small bugs and improvements)

```
/quick-fix → /staging → /release
```

| Step | Command | Expert Voice | What It Does |
|------|---------|-------------|--------------|
| 1 | `/quick-fix` | Pragmatic Developer | Skips brainstorm/planning phase. Creates ticket on Azure DevOps, branches from develop, implements fix with tests, merges to develop |
| 2 | `/staging` | QA Engineer | Same as full feature track |
| 3 | `/release` | Release Manager | Same as full feature track |

### 3. Hotfix Track (production emergencies)

```
/hotfix
```

| Step | Command | Expert Voice | What It Does |
|------|---------|-------------|--------------|
| 1 | `/hotfix` | On-call SRE / DevOps | Branches from master, fixes critical issue, runs minimal tests (unit + specific fix verification), merges to both master and develop, triggers production pipeline, creates hotfix tag |

## Command Specifications

### Bootstrap & Validation

#### `/init-project` (NEW) — Platform Engineer

Initializes a project for the full command suite.

**Actions:**
1. Read and validate `.env.claude`
2. Create git branches if missing (`develop`, `staging`)
3. Scaffold `docs/wiki/` with template files (see Knowledge Management section)
4. Generate `azure-pipelines.yml` tailored to `DEPLOY_TARGET` from `.env.claude`
5. Append wiki references to `CLAUDE.md`
6. Validate Azure DevOps PAT token connectivity

#### `/validate-env` (NEW) — DevOps Auditor

Runs a diagnostic checklist and reports pass/fail with remediation suggestions.

**Checks:**
- `.env.claude` exists and has all required fields
- PAT token authenticates successfully against Azure DevOps
- Git branches exist: `develop`, `staging`, `master`
- Azure DevOps pipelines exist (if pipeline IDs are set)
- Wiki files are populated (not just templates)
- `CLAUDE.md` has wiki references

**Output format:**
```
[PASS] .env.claude exists with required fields
[PASS] Azure DevOps PAT token valid
[FAIL] Branch 'staging' does not exist → Run /init-project to create it
[WARN] docs/wiki/api-reference.md is still a template → Run /wiki to populate it
```

### Lifecycle Commands

#### `/feature` (ENHANCED) — Business Analyst + Solution Architect

Existing command enhanced for portability via `.env.claude`.

**Skills used:** `superpowers:brainstorming`, `superpowers:writing-plans`

**Produces:** Plan file in `docs/superpowers/plans/`, spec file in `docs/superpowers/specs/`, Azure DevOps work items.

#### `/develop` (EXISTS) — Senior Developer

Existing command, no changes needed beyond `.env.claude` awareness.

**Skills used:** `superpowers:test-driven-development`, `superpowers:requesting-code-review`, `superpowers:verification-before-completion`, `superpowers:finishing-a-development-branch`

#### `/quick-fix` (NEW) — Pragmatic Developer

Streamlined path for small bugs and improvements that don't need a brainstorm/planning phase.

**Flow:**
1. Create Azure DevOps ticket with description
2. Branch from `develop` (e.g., `fix/ticket-id-short-description`)
3. Implement fix with tests
4. Verify changes
5. Request code review
6. Merge to `develop`

**Skills used:** `superpowers:test-driven-development`, `superpowers:verification-before-completion`, `superpowers:requesting-code-review`, `superpowers:finishing-a-development-branch`

#### `/hotfix` (NEW) — On-call SRE / DevOps

Emergency production fix with minimal ceremony.

**Flow:**
1. Branch from `master` (e.g., `hotfix/ticket-id-short-description`)
2. Use systematic debugging for root cause analysis
3. Implement fix with targeted tests (unit + specific fix verification)
4. Merge to both `master` and `develop`
5. Trigger production pipeline
6. Create hotfix tag (e.g., `v1.2.1-hotfix.1`)
7. Verify production deployment

**Skills used:** `superpowers:systematic-debugging`, `superpowers:verification-before-completion`

#### `/staging` (NEW) — QA Engineer

Promotes code to the staging environment and verifies deployment.

**Flow:**
1. Merge `develop` into `staging` branch
2. Push `staging` branch (triggers staging pipeline via `azure-pipelines.yml`)
3. Wait for pipeline completion (poll Azure DevOps API)
4. Run E2E tests against staging URL from `.env.claude`
5. Report deployment status and test results

**Skills used:** `superpowers:verification-before-completion`, `superpowers:dispatching-parallel-agents`

#### `/release` (NEW) — Release Manager

Promotes staging to production.

**Flow:**
1. Merge `staging` into `master`
2. Create git tag (e.g., `v1.3.0`)
3. Push `master` branch and tag (triggers production pipeline)
4. Wait for pipeline completion
5. Verify production deployment at production URL from `.env.claude`
6. Report final release status

**Skills used:** `superpowers:verification-before-completion`

### Knowledge Management

#### `/wiki` (NEW) — Technical Writer

Manages the project wiki in `docs/wiki/`.

**Subcommands:**
- `/wiki status` — Show which sections are populated vs. still templates
- `/wiki update <section>` — Analyze codebase and update a specific section
- `/wiki auto` — Auto-generate all sections from code analysis (e.g., scan API routes for `api-reference.md`, scan test files for `testing.md`)

## Knowledge Management: `docs/wiki/`

### Template Structure

Scaffolded by `/init-project`. Standard sections — fill what's relevant to the project.

| File | Purpose |
|------|---------|
| `index.md` | Table of contents + project overview (always loaded by Claude) |
| `architecture.md` | Patterns, layers, tech stack decisions |
| `api-reference.md` | Endpoints, auth, error handling |
| `data-model.md` | Database schema, entities, migrations |
| `services.md` | Background services, integrations |
| `testing.md` | Strategy, running tests, coverage targets |
| `deployment.md` | Docker, CI/CD, environments |
| `configuration.md` | Env vars, appsettings, secrets |
| `conventions.md` | Naming, patterns, coding standards |

### `index.md` Template

```markdown
# [Project Name] Developer Wiki

> Auto-generated by /init-project. Last updated: YYYY-MM-DD

## Quick Reference
- Tech Stack: [from .env.claude]
- Deploy Target: [from .env.claude]
- Staging: [URL]
- Production: [URL]

## Sections
- [Architecture](architecture.md)
- [API Reference](api-reference.md)
- [Data Model](data-model.md)
- [Services](services.md)
- [Testing](testing.md)
- [Deployment](deployment.md)
- [Configuration](configuration.md)
- [Conventions](conventions.md)
```

### Section File Frontmatter

```markdown
---
title: Architecture
last_updated: YYYY-MM-DD
status: template|draft|reviewed
---
```

The `status` field tracks wiki maturity: `template` (scaffolded, not populated), `draft` (auto-generated or partially written), `reviewed` (manually verified).

## `CLAUDE.md` Integration

`/init-project` appends the following block to `CLAUDE.md`:

```markdown
## Project Wiki Reference
- Architecture decisions: see docs/wiki/architecture.md
- API endpoints: see docs/wiki/api-reference.md
- Database schema: see docs/wiki/data-model.md
- Testing strategy: see docs/wiki/testing.md
- Deployment config: see docs/wiki/deployment.md
- Coding conventions: see docs/wiki/conventions.md

When implementing features, ALWAYS check the relevant wiki section first.
When making architectural decisions, update docs/wiki/architecture.md.
After adding API endpoints, update docs/wiki/api-reference.md.
```

This ensures Claude always has context about where project knowledge lives.

## Pipeline Generation

`/init-project` generates `azure-pipelines.yml` based on the `DEPLOY_TARGET` value from `.env.claude`. Each pipeline has two stages triggered by branch merges.

### Trigger Mapping

| Branch Merge | Stage Triggered |
|-------------|----------------|
| Merge to `staging` | Staging deployment |
| Merge to `master` | Production deployment |

### Deploy Target Templates

**Hetzner** — Build, test, SSH deploy (tar+ssh pattern):
- Build application artifact
- Run unit and integration tests
- Transfer artifact via tar+ssh to Hetzner server
- Restart application service

**Azure** — Build, test, Azure App Service deploy:
- Build application (container or code package)
- Run unit and integration tests
- Deploy to Azure App Service via Azure CLI

**Vercel** — Build, test, Vercel CLI deploy:
- Run unit and integration tests
- Deploy via Vercel CLI (or rely on Vercel Git integration for auto-deploy)

## Branching Strategy: Gitflow

```
master ──────────────────────────────── (production)
  ↑                        ↑
staging ──────────────────── (QA)
  ↑              ↑
develop ──────── (integration)
  ↑    ↑
feature/  fix/   (work branches)
```

- `develop` — integration branch, all feature and fix branches merge here
- `staging` — QA branch, promoted from develop by `/staging`
- `master` — production branch, promoted from staging by `/release`
- `hotfix/*` — branches from master, merges to both master and develop

## Skill Orchestration

Every command orchestrates existing skills. No command reimplements functionality that a skill already provides.

| Skill | Used By |
|-------|---------|
| `superpowers:brainstorming` | `/feature` |
| `superpowers:writing-plans` | `/feature` |
| `superpowers:test-driven-development` | `/develop`, `/quick-fix` |
| `superpowers:systematic-debugging` | `/hotfix` |
| `superpowers:verification-before-completion` | All lifecycle commands |
| `superpowers:requesting-code-review` | `/develop`, `/quick-fix` |
| `superpowers:finishing-a-development-branch` | `/develop`, `/quick-fix` |
| `superpowers:dispatching-parallel-agents` | `/staging` (parallel E2E tests) |
| `superpowers:executing-plans` | `/feature` (plan execution) |

## Design Principles

1. **Orchestrate, don't reinvent.** Commands compose existing skills. A new command should never duplicate logic that a skill already handles.

2. **Expert perspective in prompts, not separate agents.** Each command's system prompt is written from the perspective of the relevant domain expert (Business Analyst, QA Engineer, SRE, etc.). This shapes the reasoning and output quality without requiring separate agent infrastructure.

3. **File-driven state.** Commands produce files consumed by downstream commands. Plans live in `docs/superpowers/plans/`, specs in `docs/superpowers/specs/`, wiki content in `docs/wiki/`. This makes the pipeline observable and debuggable.

4. **Portable by design.** `.env.claude` is the only project-specific configuration. All commands read from it. Drop the skills folder plus `.env.claude` into any repository and the suite works.

5. **Gitflow branching.** `develop` (integration) → `staging` (QA) → `master` (production). Feature branches from develop. Hotfix branches from master.

6. **Centralized state machine.** `.state.md` at the project root tracks the current track, step, branch, ticket ID, blockers, and next command. Every lifecycle command reads it first and writes it last. Inspired by GSD's `.planning/STATE.md` pattern. Enables session continuity, crash recovery, and auto-advance via `/next`.

7. **Tool restrictions via frontmatter.** Each command declares `allowed-tools` in YAML frontmatter, preventing commands from acting outside their scope (e.g., `/validate-env` is read-only, `/staging` can't write code).

8. **Context rot prevention.** Long-running workflows (like `/develop`) use fresh sub-agents per task via `superpowers:dispatching-parallel-agents`, keeping the main context window lean. Each sub-agent gets a clean 200K-token context.

## State Machine: `.state.md`

Every lifecycle command reads `.state.md` at the project root before acting and writes it after completing. This is the single "where am I?" file for the entire workflow.

### Fields

| Field | Purpose |
|-------|---------|
| `track` | Current pipeline: `feature`, `quick-fix`, `hotfix`, or `idle` |
| `step` | Last completed step in the track |
| `branch` | Active working branch name |
| `ticket_id` | Azure DevOps work item ID |
| `status` | `idle`, `in-progress`, `blocked`, `awaiting-review`, `ready-to-promote` |
| `blockers` | What's blocking progress, or `none` |
| `next_command` | The command that should run next |

### Auto-Advance: `/next`

The `/next` command reads `.state.md` and automatically invokes the next command in the current track. This enables zero-touch workflow progression.

## Tracking & Measurement

Use the skill-creator eval system to track effectiveness:

| Metric | How Measured |
|--------|-------------|
| Command invocation success rate | Track pass/fail per command execution |
| Full cycle time | Time from `/feature` to `/release` completion |
| Validation catch rate | How often `/validate-env` catches issues before they block work |
| Wiki coverage | Percentage of sections with status `draft` or `reviewed` vs. `template` |

## Command Summary

| Command | Status | Track | Expert Voice | Key Skills Used |
|---------|--------|-------|-------------|----------------|
| `/init-project` | NEW | Bootstrap | Platform Engineer | — |
| `/validate-env` | NEW | Bootstrap | DevOps Auditor | — |
| `/next` | NEW | All Tracks | Workflow Orchestrator | reads .state.md, invokes next command |
| `/feature` | ENHANCED | Full | Business Analyst + Architect | brainstorming, writing-plans |
| `/develop` | EXISTS | Full, Quick | Senior Developer | TDD, code-review, verification |
| `/quick-fix` | NEW | Quick | Pragmatic Developer | TDD, verification, code-review |
| `/hotfix` | NEW | Hotfix | On-call SRE | systematic-debugging, verification |
| `/staging` | NEW | Full, Quick | QA Engineer | verification, dispatching-parallel-agents |
| `/release` | NEW | Full, Quick, Hotfix | Release Manager | verification |
| `/wiki` | NEW | Maintenance | Technical Writer | — |
| `/board` | ENHANCED | Info | — | — |
| `/overview` | ENHANCED | Info | — | — |
| `/ticket` | ENHANCED | Info | — | — |

## Implementation Priority

1. **Phase 1 — Foundation:** `.env.claude` schema, `/init-project`, `/validate-env`, wiki scaffolding
2. **Phase 2 — Lifecycle commands:** `/staging`, `/release`, `/quick-fix`, `/hotfix`
3. **Phase 3 — Portability:** Enhance existing commands (`/board`, `/overview`, `/ticket`) to read from `.env.claude`
4. **Phase 4 — Knowledge:** `/wiki` command with auto-population from codebase analysis
5. **Phase 5 — Measurement:** Eval tracking integration
