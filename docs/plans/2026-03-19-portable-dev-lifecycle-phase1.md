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
| `commands/init-project.md` | Create | Bootstrap command: scaffold wiki, generate pipeline, update CLAUDE.md |
| `commands/validate-env.md` | Create | Health check command: verify repo setup is complete and correct |
| `templates/.env.claude.example` | Exists | Reference schema (already created) |
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

## Task 2: Create CLAUDE.md Wiki Reference Template

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

## Task 3: Create `/init-project` Command

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

## Step 8: Commit

Commit all generated files:

```bash
git add docs/wiki/ azure-pipelines.yml CLAUDE.md
git commit -m "chore: bootstrap project with init-project (wiki, pipeline, CLAUDE.md)"
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

## Task 4: Create `/validate-env` Command

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

## Task 5: Create `.env.claude` for MDT Dynamics

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

## Task 6: Backport Portability to MDT Dynamics Commands

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

## Task 7: Verify Backported Commands Work

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

## Task 8: Push Claude Skills Repo Updates

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

After Phase 1 completion (8 tasks):

| Task | Deliverable | Repo |
|------|-------------|------|
| 1 | Shared config loader (`commands/_shared/load-config.md`) | Claude Skills |
| 2 | CLAUDE.md wiki block template (`templates/claude-md-wiki-block.md`) | Claude Skills |
| 3 | `/init-project` command (`commands/init-project.md`) | Claude Skills |
| 4 | `/validate-env` command (`commands/validate-env.md`) | Claude Skills |
| 5 | `.env.claude` for MDT Dynamics + `.gitignore` update | MDT Dynamics |
| 6 | 5 commands backported to portable config (board, overview, ticket, update-ticket, create-backlog) | MDT Dynamics |
| 7 | End-to-end verification of portable config pattern | MDT Dynamics |
| 8 | Push all Claude Skills changes to Azure DevOps | Claude Skills |

**Next:** Phase 2 — Lifecycle commands (`/staging`, `/release`, `/quick-fix`, `/hotfix`)
