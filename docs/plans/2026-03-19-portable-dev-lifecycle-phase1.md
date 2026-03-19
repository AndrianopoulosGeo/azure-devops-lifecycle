# Phase 1: Foundation — `.env.claude`, `/init-project`, `/validate-env`, Wiki Scaffolding

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the foundation layer — config schema, bootstrap command, validation command, and wiki templates — so all other commands can build on top of portable `.env.claude` configuration.

**Architecture:** Three new files form the foundation: a shared config-loading snippet (sourced by all commands), `/init-project` that scaffolds a project from `.env.claude`, and `/validate-env` that health-checks the setup. Wiki templates from `templates/wiki/` are copied and hydrated with project values during init.

**Tech Stack:** Bash (Azure CLI, git), Claude Code commands (markdown prompts), Azure DevOps REST API via `az` CLI

**Working Directory:** `C:\Users\andri\Dev\Claude Skills`

**Scope Note:** This plan intentionally pulls forward the "backport existing commands" work from Phase 3 into Phase 1. Reason: we need at least one real project (MDT Dynamics) using `.env.claude` to validate the config pattern end-to-end before building lifecycle commands in Phase 2. This is a deliberate scope expansion, not a spec deviation.

---

## File Structure

| File | Action | Responsibility |
|------|--------|---------------|
| `commands/_shared/load-config.md` | Create | Shared config-loading instructions snippet sourced by all commands |
| `commands/_shared/state-management.md` | Create | Shared state machine pattern — read/write `.state.md` |
| `commands/init-project.md` | Create | Bootstrap command: scaffold wiki, generate pipeline, update CLAUDE.md |
| `commands/validate-env.md` | Create | Health check command: verify repo setup is complete and correct |
| `commands/next.md` | Create | Auto-advance command — reads `.state.md` and runs next step in track |
| `templates/.env.claude.example` | Exists | Reference schema (already created) |
| `templates/.state.md.example` | Create | State machine template with all fields |
| `templates/wiki/*.md` | Exists | Wiki templates (already created, 9 files) |
| `templates/pipelines/*.yml` | Exists | Pipeline templates (already created, 3 files) |
| `templates/claude-md-wiki-block.md` | Create | CLAUDE.md snippet template for wiki references |
| `docs/design-spec.md` | Exists | Design specification (already committed) |

---

## Task 1: Create Shared Config Loading Pattern

**Files:**
- Create: `commands/_shared/load-config.md`

This is a reusable prompt snippet that every command will reference. It defines how to read `.env.claude` and set up Azure DevOps CLI context.

- [ ] **Step 1: Create the shared config loader**

```markdown
## Configuration Loading (source this in every command)

Before any Azure DevOps operation, load project configuration:

1. Read `.env.claude` from the repository root:
   ```bash
   # Load all config from .env.claude
   export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
   export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
   export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
   export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
   export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
   export STAGING_URL=$(grep STAGING_URL .env.claude | cut -d '=' -f2)
   export PRODUCTION_URL=$(grep PRODUCTION_URL .env.claude | cut -d '=' -f2)
   export BRANCH_STRATEGY=$(grep BRANCH_STRATEGY .env.claude | cut -d '=' -f2)
   export STAGING_PIPELINE_ID=$(grep STAGING_PIPELINE_ID .env.claude | cut -d '=' -f2)
   export PRODUCTION_PIPELINE_ID=$(grep PRODUCTION_PIPELINE_ID .env.claude | cut -d '=' -f2)
   ```

2. Configure Azure DevOps CLI defaults:
   ```bash
   export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
   az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
   ```

3. If `.env.claude` does not exist, stop and tell the user:
   > ".env.claude not found. Create one from the template: `cp templates/.env.claude.example .env.claude` and fill in your values. Then run `/init-project`."

### Required Fields
These fields MUST be present and non-empty: `AZURE_DEVOPS_ORG`, `AZURE_DEVOPS_PROJECT`, `AZURE_DEVOPS_PAT`, `DEPLOY_TARGET`, `TECH_STACK`.

### Optional Fields
These fields may be empty: `STAGING_URL`, `PRODUCTION_URL`, `STAGING_PIPELINE_ID`, `PRODUCTION_PIPELINE_ID`, `BRANCH_STRATEGY` (defaults to `gitflow`).

### Deploy-Target-Specific Fields
Additional fields (e.g., `SSH_HOST`, `SSH_USER`, `AZURE_SUBSCRIPTION_ID`, `VERCEL_TOKEN`) are loaded by individual commands (`/staging`, `/release`, `/hotfix`) as needed. The shared loader only covers the universal fields above.
```

Write this to `commands/_shared/load-config.md`.

- [ ] **Step 2: Verify file was created**

Run: `cat commands/_shared/load-config.md | head -5`
Expected: The header lines of the config loading pattern.

- [ ] **Step 3: Commit**

```bash
git add commands/_shared/load-config.md
git commit -m "feat(config): add shared config loading pattern for .env.claude"
```

---

## Task 2: Create State Machine Pattern and Template

**Files:**
- Create: `commands/_shared/state-management.md`
- Create: `templates/.state.md.example`

The state machine is inspired by GSD's `.planning/STATE.md` pattern. Every lifecycle command reads `.state.md` first (to know where we are) and writes it last (to record what happened). This enables session continuity, crash recovery, and auto-advance via `/next`.

- [ ] **Step 1: Create the state template**

Write to `templates/.state.md.example`:

```markdown
# Project State

> Auto-managed by lifecycle commands. Do not edit manually unless recovering from a failure.

## Current Work

| Field | Value |
|-------|-------|
| track | — |
| step | — |
| branch | — |
| ticket_id | — |
| started_at | — |
| last_command | — |
| last_updated | — |

## Status

| Field | Value |
|-------|-------|
| status | idle |
| blockers | none |
| next_command | — |

## History

<!-- Appended by each command -->
```

**Field definitions:**
- `track`: `feature`, `quick-fix`, `hotfix`, or `idle`
- `step`: Current step in the track (e.g., `develop`, `staging`, `release`)
- `branch`: Active working branch name
- `ticket_id`: Azure DevOps work item ID being worked on
- `started_at`: ISO timestamp when the track started
- `last_command`: Last command that ran (e.g., `/develop`)
- `last_updated`: ISO timestamp of last state change
- `status`: `idle`, `in-progress`, `blocked`, `awaiting-review`, `ready-to-promote`
- `blockers`: Description of what's blocking, or `none`
- `next_command`: The command that should run next (used by `/next`)

- [ ] **Step 2: Create the shared state management instructions**

Write to `commands/_shared/state-management.md`:

```markdown
## State Management (source this in every lifecycle command)

### Reading State

At the START of every lifecycle command (`/feature`, `/develop`, `/quick-fix`, `/hotfix`, `/staging`, `/release`):

1. Check if `.state.md` exists at the repository root
2. If it exists, read the current state:
   ```bash
   # Quick state check
   grep -A1 "track" .state.md | tail -1 | awk '{print $NF}'
   grep -A1 "step" .state.md | tail -1 | awk '{print $NF}'
   grep -A1 "status" .state.md | tail -1 | awk '{print $NF}'
   ```
3. If the state shows a different track is in progress, WARN the user:
   > "There's already a [track] in progress on branch [branch] (ticket #[id]). Do you want to continue that work, or start fresh?"
4. If `.state.md` doesn't exist, create it from the template with status `idle`

### Writing State

At the END of every lifecycle command, update `.state.md`:

1. Update the `Current Work` table with current values
2. Update `Status` table — set `status`, clear/set `blockers`, set `next_command`
3. Append a line to the `History` section:
   ```markdown
   - [YYYY-MM-DD HH:MM] `/command-name` — status: result (brief summary)
   ```

### Track → Step → Next Command Mapping

| Track | Step Sequence | Next Command |
|-------|--------------|-------------|
| feature | feature → develop → staging → release → idle | `/develop` → `/staging` → `/release` → done |
| quick-fix | quick-fix → staging → release → idle | `/staging` → `/release` → done |
| hotfix | hotfix → idle | done |

### Status Values

| Status | Meaning | Allowed Next Actions |
|--------|---------|---------------------|
| `idle` | No work in progress | Start any track |
| `in-progress` | Command is currently running | Wait or resume |
| `blocked` | Cannot proceed, needs intervention | Fix blocker, then `/next` |
| `awaiting-review` | Code review or approval needed | Review, then `/next` |
| `ready-to-promote` | Ready for next environment | `/staging` or `/release` |
```

- [ ] **Step 3: Verify both files created**

Run: `ls -la commands/_shared/ templates/.state.md.example`
Expected: Both files listed.

- [ ] **Step 4: Commit**

```bash
git add commands/_shared/state-management.md templates/.state.md.example
git commit -m "feat(state): add centralized state machine pattern (.state.md)"
```

---

## Task 3: Create `/next` Auto-Advance Command

**Files:**
- Create: `commands/next.md`

This command reads `.state.md` and automatically runs the next command in the current track. Inspired by GSD's `/gsd:next`.

- [ ] **Step 1: Write the `/next` command**

Write to `commands/next.md`:

```markdown
---
allowed-tools: [Read, Bash, Skill]
---

# /next — Auto-Advance to Next Step

> **Expert Voice:** Workflow Orchestrator — reads state, determines next action, hands off to the right command.

You are a workflow orchestrator. Your ONLY job is to read the current project state and invoke the next command in the pipeline. You do NOT implement anything yourself.

## Step 1: Read State

Read `.state.md` from the repository root:

```bash
cat .state.md
```

If `.state.md` does not exist:
> "No active workflow. Start one with `/feature`, `/quick-fix`, or `/hotfix`."
Stop.

## Step 2: Determine Next Action

Parse the state fields:
- `track`: Which pipeline are we in?
- `step`: What was the last completed step?
- `status`: Are we blocked or ready?
- `next_command`: What should run next?

### Decision Matrix

| Status | Action |
|--------|--------|
| `idle` | "No active workflow. Start one with `/feature`, `/quick-fix`, or `/hotfix`." — Stop. |
| `blocked` | "Workflow is blocked: [blockers]. Resolve the issue and run `/next` again." — Stop. |
| `awaiting-review` | "Step [step] is awaiting review. Complete the review and run `/next` again." — Stop. |
| `in-progress` | "Step [step] is still in progress. Let it complete or resume with `/[step]`." — Stop. |
| `ready-to-promote` | Read `next_command` and invoke it. |

### Track Routing

If `next_command` is set and status is `ready-to-promote`:

| next_command | Action |
|-------------|--------|
| `/develop` | Invoke the Skill tool with skill: "develop" |
| `/staging` | Invoke the Skill tool with skill: "staging" |
| `/release` | Invoke the Skill tool with skill: "release" |
| `done` | Report: "Workflow complete! Track: [track], Ticket: #[ticket_id]. State reset to idle." Then update `.state.md` to idle. |

## Step 3: Report

Before invoking the next command, print:
```
/next — Auto-advancing workflow
  Track: [track]
  Completed: [step]
  Next: [next_command]
  Branch: [branch]
  Ticket: #[ticket_id]
```

Then invoke the next command.
```

- [ ] **Step 2: Verify the file**

Run: `wc -l commands/next.md`
Expected: ~70-80 lines

- [ ] **Step 3: Commit**

```bash
git add commands/next.md
git commit -m "feat(commands): add /next auto-advance command"
```

---

## Task 4: Add `allowed-tools` Frontmatter to Commands

**Files:**
- Modify: `commands/init-project.md` (after Task 5 creates it)
- Modify: `commands/validate-env.md` (after Task 6 creates it)
- Modify: `commands/next.md` (already has it from Task 3)

This is applied AFTER the commands are created (Tasks 5-6). Each command gets YAML frontmatter restricting which tools it can use. This prevents commands from doing things outside their scope (e.g., `/validate-env` shouldn't write code).

- [ ] **Step 1: Add frontmatter to `/init-project`**

Add to the top of `commands/init-project.md`:
```yaml
---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
---
```

Rationale: `/init-project` needs to create files (Write), modify CLAUDE.md (Edit), run git/az CLI (Bash), and find existing files (Glob/Grep). It does NOT need WebSearch, Agent, or Skill.

- [ ] **Step 2: Add frontmatter to `/validate-env`**

Add to the top of `commands/validate-env.md`:
```yaml
---
allowed-tools: [Read, Bash, Glob, Grep]
---
```

Rationale: `/validate-env` is read-only — it checks things but never modifies them. No Write, Edit, Agent, or Skill.

- [ ] **Step 3: Verify all three commands have frontmatter**

```bash
head -3 commands/init-project.md commands/validate-env.md commands/next.md
```

Expected: Each file starts with `---` followed by `allowed-tools:`.

- [ ] **Step 4: Commit**

```bash
git add commands/init-project.md commands/validate-env.md
git commit -m "feat(commands): add allowed-tools frontmatter for tool restrictions"
```

**Note:** This task depends on Tasks 5 and 6 completing first. If using subagent-driven execution, sequence accordingly.

---

## Task 5: Create CLAUDE.md Wiki Reference Template

**Files:**
- Create: `templates/claude-md-wiki-block.md`

This template block gets appended to a project's CLAUDE.md by `/init-project`.

- [ ] **Step 1: Create the template**

```markdown
## Project Wiki Reference

The project wiki at `docs/wiki/` is the single source of truth for developer knowledge.

- Architecture decisions: see `docs/wiki/architecture.md`
- API endpoints: see `docs/wiki/api-reference.md`
- Database schema: see `docs/wiki/data-model.md`
- Background services: see `docs/wiki/services.md`
- Testing strategy: see `docs/wiki/testing.md`
- Deployment config: see `docs/wiki/deployment.md`
- Environment variables: see `docs/wiki/configuration.md`
- Coding conventions: see `docs/wiki/conventions.md`

**Rules:**
- When implementing features, ALWAYS check the relevant wiki section first.
- When making architectural decisions, update `docs/wiki/architecture.md`.
- After adding API endpoints, update `docs/wiki/api-reference.md`.
- After changing database schema, update `docs/wiki/data-model.md`.
- After modifying tests, update `docs/wiki/testing.md`.
```

Write this to `templates/claude-md-wiki-block.md`.

- [ ] **Step 2: Commit**

```bash
git add templates/claude-md-wiki-block.md
git commit -m "feat(templates): add CLAUDE.md wiki reference block template"
```

---

## Task 6: Create `/init-project` Command

**Files:**
- Create: `commands/init-project.md`

This is the main bootstrap command. It reads `.env.claude`, scaffolds the wiki, generates pipeline YAML, and updates CLAUDE.md.

- [ ] **Step 1: Write the init-project command**

The command should follow the exact prompt pattern used by existing commands (e.g., `board.md`, `feature.md`). Write the full command to `commands/init-project.md` with this structure:

```markdown
# /init-project — Project Bootstrap

> **Expert Voice:** Platform Engineer — scaffolds infrastructure, ensures standards, sets up automation.

You are a Platform Engineer bootstrapping a new project for the portable development lifecycle. Your job is to read `.env.claude` and set up everything the team needs: wiki, pipelines, branches, and CLAUDE.md references.

## Prerequisites

Before starting, verify:
1. `.env.claude` exists at the repository root
2. Git is initialized in this directory
3. The Azure CLI (`az`) is installed and available

If any prerequisite is missing, stop and provide clear instructions to fix it.

## Step 1: Load Configuration

Load all configuration from `.env.claude`:

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
export STAGING_URL=$(grep STAGING_URL .env.claude | cut -d '=' -f2)
export PRODUCTION_URL=$(grep PRODUCTION_URL .env.claude | cut -d '=' -f2)
export BRANCH_STRATEGY=$(grep BRANCH_STRATEGY .env.claude | cut -d '=' -f2)
```

Validate that all required fields are present and non-empty: `AZURE_DEVOPS_ORG`, `AZURE_DEVOPS_PROJECT`, `AZURE_DEVOPS_PAT`, `DEPLOY_TARGET`, `TECH_STACK`.

If any required field is missing, print exactly which fields are missing and stop.

## Step 2: Validate Azure DevOps Connection

```bash
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
az devops project show --project "$AZURE_DEVOPS_PROJECT" --output table
```

If this fails, print the error and suggest checking the PAT token permissions (needs Work Items Read/Write, Code Read/Write, Build Read/Execute).

## Step 3: Create Git Branches

Check which branches exist and create any that are missing:

```bash
# Check existing branches
git branch -a

# Create develop if missing
git rev-parse --verify develop 2>/dev/null || git branch develop

# Create staging if missing
git rev-parse --verify staging 2>/dev/null || git branch staging
```

Report which branches already existed and which were created.

## Step 4: Scaffold Wiki

Create `docs/wiki/` directory and copy template files. Replace template placeholders with values from `.env.claude`:

For each wiki template file in `templates/wiki/`:
1. Copy to `docs/wiki/`
2. Replace `{{PROJECT_NAME}}` with `$AZURE_DEVOPS_PROJECT`
3. Replace `{{TECH_STACK}}` with `$TECH_STACK`
4. Replace `{{DEPLOY_TARGET}}` with `$DEPLOY_TARGET`
5. Replace `{{STAGING_URL}}` with `$STAGING_URL`
6. Replace `{{PRODUCTION_URL}}` with `$PRODUCTION_URL`
7. Replace `{{AZURE_DEVOPS_ORG}}` with `$AZURE_DEVOPS_ORG`
8. Replace `{{AZURE_DEVOPS_PROJECT}}` with `$AZURE_DEVOPS_PROJECT`
9. Replace `{{DATE}}` with today's date (YYYY-MM-DD)
10. Update `last_updated` in frontmatter to today's date

If `docs/wiki/` already exists and has non-template content (status != "template"), ask the user before overwriting:
> "Wiki files already exist and appear to have content. Overwrite? (y/n)"

## Step 5: Generate Azure Pipeline

Based on `DEPLOY_TARGET` and `TECH_STACK`, select the right pipeline template:

| TECH_STACK | DEPLOY_TARGET | Template |
|-----------|--------------|----------|
| nextjs | hetzner | `templates/pipelines/azure-pipelines-nextjs-hetzner.yml` |
| nextjs | vercel | `templates/pipelines/azure-pipelines-nextjs-vercel.yml` |
| dotnet | azure | `templates/pipelines/azure-pipelines-dotnet-azure.yml` |

1. Copy the matching template to `azure-pipelines.yml` at the repository root
2. If no exact match exists, warn the user and skip pipeline generation:
   > "No pipeline template for TECH_STACK=$TECH_STACK + DEPLOY_TARGET=$DEPLOY_TARGET. You'll need to create azure-pipelines.yml manually."
3. If `azure-pipelines.yml` already exists, ask before overwriting

## Step 6: Update CLAUDE.md

Read the `templates/claude-md-wiki-block.md` template and append it to the project's `CLAUDE.md`.

1. Check if `CLAUDE.md` exists — if not, create it with a basic header first
2. Check if the wiki reference block already exists (search for "## Project Wiki Reference") — if so, skip
3. Append the wiki reference block to the end of `CLAUDE.md`

## Step 7: Summary Report

Print a summary of everything that was done:

```
/init-project completed:

Project: $AZURE_DEVOPS_PROJECT
Org:     https://dev.azure.com/$AZURE_DEVOPS_ORG
Target:  $DEPLOY_TARGET ($TECH_STACK)

[DONE] Azure DevOps connection verified
[DONE] Branch 'develop' — existed | created
[DONE] Branch 'staging' — existed | created
[DONE] Wiki scaffolded (9 files in docs/wiki/)
[DONE] Pipeline generated (azure-pipelines.yml)
[DONE] CLAUDE.md updated with wiki references

Next steps:
1. Run /validate-env to verify everything is correct
2. Populate wiki sections with /wiki auto
3. Push azure-pipelines.yml and configure pipeline in Azure DevOps
4. Start developing with /feature or /quick-fix
```

## Step 8: Initialize State File

Copy `.state.md` template from `templates/.state.md.example` to the repository root. Set all fields to their default idle values. This enables `/next` auto-advance from the first workflow.

## Step 9: Commit

Commit all generated files:

```bash
git add docs/wiki/ azure-pipelines.yml CLAUDE.md .state.md
git commit -m "chore: bootstrap project with init-project (wiki, pipeline, state, CLAUDE.md)"
```
```

Write the complete command above to `commands/init-project.md`.

- [ ] **Step 2: Verify the file structure**

Run: `wc -l commands/init-project.md`
Expected: ~130-160 lines

- [ ] **Step 3: Commit**

```bash
git add commands/init-project.md
git commit -m "feat(commands): add /init-project bootstrap command"
```

---

## Task 7: Create `/validate-env` Command

**Files:**
- Create: `commands/validate-env.md`

- [ ] **Step 1: Write the validate-env command**

Write the full command to `commands/validate-env.md`:

```markdown
# /validate-env — Environment Health Check

> **Expert Voice:** DevOps Auditor — methodical, checklist-driven, reports pass/fail with remediation.

You are a DevOps Auditor running a comprehensive health check on the project setup. Check every aspect of the configuration, report results clearly, and provide actionable remediation for any failures.

## Load Configuration

First, attempt to load `.env.claude`:

```bash
if [ ! -f .env.claude ]; then
  echo "[FAIL] .env.claude not found"
  echo "  → Create from template: cp templates/.env.claude.example .env.claude"
  exit 1
fi
```

Then load all fields:

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
export STAGING_URL=$(grep STAGING_URL .env.claude | cut -d '=' -f2)
export PRODUCTION_URL=$(grep PRODUCTION_URL .env.claude | cut -d '=' -f2)
export BRANCH_STRATEGY=$(grep BRANCH_STRATEGY .env.claude | cut -d '=' -f2)
export STAGING_PIPELINE_ID=$(grep STAGING_PIPELINE_ID .env.claude | cut -d '=' -f2)
export PRODUCTION_PIPELINE_ID=$(grep PRODUCTION_PIPELINE_ID .env.claude | cut -d '=' -f2)
```

## Checks

Run ALL checks, even if earlier ones fail. Collect all results and present them together at the end.

### 1. Configuration File Checks

| Check | Command | Pass | Fail |
|-------|---------|------|------|
| `.env.claude` exists | `test -f .env.claude` | [PASS] | [FAIL] → Create from template |
| `AZURE_DEVOPS_ORG` set | `test -n "$AZURE_DEVOPS_ORG"` | [PASS] | [FAIL] → Add to .env.claude |
| `AZURE_DEVOPS_PROJECT` set | `test -n "$AZURE_DEVOPS_PROJECT"` | [PASS] | [FAIL] → Add to .env.claude |
| `AZURE_DEVOPS_PAT` set | `test -n "$AZURE_DEVOPS_PAT"` | [PASS] | [FAIL] → Add to .env.claude |
| `DEPLOY_TARGET` valid | Value is one of: hetzner, azure, vercel | [PASS] | [FAIL] → Must be hetzner\|azure\|vercel |
| `TECH_STACK` valid | Value is one of: nextjs, dotnet, python | [PASS] | [FAIL] → Must be nextjs\|dotnet\|python |

### 2. Azure DevOps Connection

```bash
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
az devops project show --project "$AZURE_DEVOPS_PROJECT" --output table 2>&1
```

| Check | Pass | Fail |
|-------|------|------|
| PAT authenticates | [PASS] | [FAIL] → Check PAT token permissions and expiry |
| Project accessible | [PASS] | [FAIL] → Check project name matches Azure DevOps |

### 3. Git Branch Checks

```bash
git branch -a
```

| Check | Pass | Fail |
|-------|------|------|
| `master` branch exists | [PASS] | [FAIL] → Run: git branch master |
| `develop` branch exists | [PASS] | [FAIL] → Run: git branch develop |
| `staging` branch exists | [PASS] | [FAIL] → Run: git branch staging OR run /init-project |

### 4. Wiki Checks

| Check | Pass | Fail |
|-------|------|------|
| `docs/wiki/` exists | [PASS] | [FAIL] → Run /init-project |
| `docs/wiki/index.md` exists | [PASS] | [FAIL] → Run /init-project |
| Wiki has content (not just templates) | [PASS] Count: N/9 populated | [WARN] → Run /wiki auto to populate |

To check if a wiki file is populated vs template, read the `status` field in the frontmatter:
- `template` = not populated
- `draft` or `reviewed` = has content

### 5. Pipeline Checks

| Check | Pass | Fail |
|-------|------|------|
| `azure-pipelines.yml` exists | [PASS] | [FAIL] → Run /init-project |
| Pipeline matches DEPLOY_TARGET | [PASS] | [WARN] → Pipeline may be stale, regenerate with /init-project |

If `STAGING_PIPELINE_ID` is set in `.env.claude`, also verify it exists in Azure DevOps:
```bash
az pipelines show --id $STAGING_PIPELINE_ID --output table 2>&1
```
Same for `PRODUCTION_PIPELINE_ID`. If the API call fails, report:
> [FAIL] Staging pipeline ID $STAGING_PIPELINE_ID not found in Azure DevOps → Verify pipeline exists or clear STAGING_PIPELINE_ID in .env.claude

### 6. CLAUDE.md Checks

| Check | Pass | Fail |
|-------|------|------|
| `CLAUDE.md` exists | [PASS] | [FAIL] → Run /init-project |
| Wiki references present | Search for "Project Wiki Reference" | [FAIL] → Run /init-project |

### 7. .gitignore Check

| Check | Pass | Fail |
|-------|------|------|
| `.env.claude` is gitignored | `grep -q '.env.claude' .gitignore` | [WARN] → Add `.env.claude` to .gitignore to prevent secret leaks |

## Output Format

Present all results in a clean table:

```
/validate-env — Environment Health Check
==========================================
Project: $AZURE_DEVOPS_PROJECT
Org:     https://dev.azure.com/$AZURE_DEVOPS_ORG
Target:  $DEPLOY_TARGET ($TECH_STACK)

Configuration
  [PASS] .env.claude exists
  [PASS] All required fields present
  [PASS] DEPLOY_TARGET=hetzner (valid)
  [PASS] TECH_STACK=nextjs (valid)

Azure DevOps
  [PASS] PAT token authenticates
  [PASS] Project "MDT dynamics" accessible

Git Branches
  [PASS] master exists
  [PASS] develop exists
  [FAIL] staging missing → Run: git branch staging

Wiki (5/9 populated)
  [PASS] docs/wiki/ exists
  [PASS] architecture.md — reviewed
  [PASS] testing.md — draft
  [WARN] api-reference.md — template (not populated)
  [WARN] data-model.md — template (not populated)
  [WARN] services.md — template (not populated)
  [WARN] configuration.md — template (not populated)

Pipeline
  [PASS] azure-pipelines.yml exists
  [PASS] Matches DEPLOY_TARGET=hetzner

CLAUDE.md
  [PASS] CLAUDE.md exists
  [PASS] Wiki references present

Security
  [PASS] .env.claude is gitignored

==========================================
Result: 14 PASS | 1 FAIL | 4 WARN

Action Required:
  1. [FAIL] Create staging branch: git branch staging
  2. [WARN] Populate wiki sections: /wiki auto
```

If all checks pass, end with:
```
All checks passed. Environment is ready for development.
```
```

Write this to `commands/validate-env.md`.

- [ ] **Step 2: Verify the file**

Run: `wc -l commands/validate-env.md`
Expected: ~140-170 lines

- [ ] **Step 3: Commit**

```bash
git add commands/validate-env.md
git commit -m "feat(commands): add /validate-env health check command"
```

---

## Task 8: Create `.env.claude` for MDT Dynamics

**Working Directory:** `C:\Users\andri\Dev\MDT dynamics`

**Files:**
- Create: `C:\Users\andri\Dev\MDT dynamics\.env.claude`
- Modify: `C:\Users\andri\Dev\MDT dynamics\.gitignore`

This task sets up the first real project to use `.env.claude`, validating the config pattern end-to-end.

- [ ] **Step 1: Read existing PAT from `.env`**

```bash
grep AZURE_DEVOPS_PAT .env | cut -d '=' -f2
```

Do NOT hardcode the PAT — read it from the existing file.

- [ ] **Step 2: Create `.env.claude`**

Create `.env.claude` at `C:\Users\andri\Dev\MDT dynamics\.env.claude` with:

```env
# Azure DevOps
AZURE_DEVOPS_ORG=pgSquare
AZURE_DEVOPS_PROJECT=MDT dynamics
AZURE_DEVOPS_PAT=<value from Step 1>

# Deploy target
DEPLOY_TARGET=hetzner
TECH_STACK=nextjs

# Environment URLs
STAGING_URL=
PRODUCTION_URL=

# Branch strategy
BRANCH_STRATEGY=gitflow

# Pipeline IDs (to be created)
STAGING_PIPELINE_ID=
PRODUCTION_PIPELINE_ID=
```

- [ ] **Step 3: Add `.env.claude` to `.gitignore`**

Check if `.env.claude` is already in `.gitignore`. If not, add it alongside the existing `.env` entry.

- [ ] **Step 4: Commit**

```bash
git add .env.claude .gitignore
git commit -m "chore: add .env.claude config for portable command suite"
```

**Note:** `.env.claude` contains secrets — verify `.gitignore` excludes it BEFORE committing. If `.gitignore` is set up correctly, `git add .env.claude` will be rejected, which is expected. In that case, just commit `.gitignore`.

---

## Task 9: Backport Portability to MDT Dynamics Commands

**Working Directory:** `C:\Users\andri\Dev\MDT dynamics`

**Files:**
- Modify: `C:\Users\andri\Dev\MDT dynamics\.claude\commands\board.md`
- Modify: `C:\Users\andri\Dev\MDT dynamics\.claude\commands\overview.md`
- Modify: `C:\Users\andri\Dev\MDT dynamics\.claude\commands\ticket.md`
- Modify: `C:\Users\andri\Dev\MDT dynamics\.claude\commands\update-ticket.md`
- Modify: `C:\Users\andri\Dev\MDT dynamics\.claude\commands\create-backlog.md`

Update 5 existing commands to read from `.env.claude` instead of hardcoded values.

- [ ] **Step 1: Update `board.md` config loading**

Replace the hardcoded config pattern:
```bash
export AZURE_DEVOPS_EXT_PAT=$(grep AZURE_DEVOPS_PAT .env | cut -d '=' -f2)
az devops configure --defaults organization=https://dev.azure.com/pgSquare project="MDT dynamics"
```

With the portable pattern:
```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

Also replace all hardcoded `'MDT dynamics'` in WIQL queries with `'$AZURE_DEVOPS_PROJECT'` (using double quotes for bash variable expansion).

- [ ] **Step 2: Update `overview.md` config loading**

Same pattern replacement as board.md. Replace hardcoded `.env` reads and `'MDT dynamics'` WIQL references.

- [ ] **Step 3: Update `ticket.md` config loading**

Same pattern replacement.

- [ ] **Step 4: Update `update-ticket.md` config loading**

Same pattern replacement.

- [ ] **Step 5: Update `create-backlog.md` config loading**

Same pattern replacement. This command also has hardcoded org/project in WIQL queries — replace all instances.

- [ ] **Step 6: Commit**

```bash
git add .claude/commands/board.md .claude/commands/overview.md .claude/commands/ticket.md .claude/commands/update-ticket.md .claude/commands/create-backlog.md
git commit -m "refactor(commands): make 5 commands portable via .env.claude"
```

---

## Task 10: Verify Backported Commands Work

**Working Directory:** `C:\Users\andri\Dev\MDT dynamics`

- [ ] **Step 1: Test the portable config loading**

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
az boards query --wiql "SELECT [System.Id], [System.Title] FROM workitems WHERE [System.TeamProject] = '$AZURE_DEVOPS_PROJECT' AND [System.State] = 'Active'" --output table
```

Expected: Active work items displayed (same results as before the refactor).

- [ ] **Step 2: Verify variable expansion in WIQL**

Confirm that `$AZURE_DEVOPS_PROJECT` expands correctly to `MDT dynamics` in the query (check the output includes expected work items, not an empty result from a mismatched project name).

---

## Task 11: Push Claude Skills Repo Updates

**Working Directory:** `C:\Users\andri\Dev\Claude Skills`

- [ ] **Step 1: Verify current branch**

```bash
git branch --show-current
```

Expected: `master`

- [ ] **Step 2: Push all Phase 1 commits**

```bash
git push origin master
```

- [ ] **Step 3: Verify on Azure DevOps**

Confirm the push succeeded and all files are visible in the Azure DevOps repo.

---

## Summary

After Phase 1 completion (11 tasks):

| Task | Deliverable | Repo |
|------|-------------|------|
| 1 | Shared config loader (`commands/_shared/load-config.md`) | Claude Skills |
| 2 | State machine pattern + template (`commands/_shared/state-management.md`, `templates/.state.md.example`) | Claude Skills |
| 3 | `/next` auto-advance command (`commands/next.md`) | Claude Skills |
| 4 | `allowed-tools` frontmatter on all commands | Claude Skills |
| 5 | CLAUDE.md wiki block template (`templates/claude-md-wiki-block.md`) | Claude Skills |
| 6 | `/init-project` command (`commands/init-project.md`) | Claude Skills |
| 7 | `/validate-env` command (`commands/validate-env.md`) | Claude Skills |
| 8 | `.env.claude` for MDT Dynamics + `.gitignore` update | MDT Dynamics |
| 9 | 5 commands backported to portable config (board, overview, ticket, update-ticket, create-backlog) | MDT Dynamics |
| 10 | End-to-end verification of portable config pattern | MDT Dynamics |
| 11 | Push all Claude Skills changes to Azure DevOps | Claude Skills |

**Next:** Phase 2 — Lifecycle commands (`/staging`, `/release`, `/quick-fix`, `/hotfix`)
