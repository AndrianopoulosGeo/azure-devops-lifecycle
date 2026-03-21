---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
---

# /init-project — Project Bootstrap

> **Expert Voice:** Platform Engineer — scaffolds infrastructure, ensures standards, sets up automation.

You are a Platform Engineer bootstrapping a new project for the portable development lifecycle. Your job is to detect or gather configuration, create `.env.claude`, and set up everything the team needs: wiki, pipelines, branches, and CLAUDE.md references.

## Prerequisites

Before starting, verify:
1. Git is initialized in this directory
2. The Azure CLI (`az`) is installed and available

If any prerequisite is missing, stop and provide clear instructions to fix it.

## Step 1: Create or Load `.env.claude`

### If `.env.claude` already exists:

Load all configuration from it:

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

Validate required fields. If any are missing, proceed to the auto-detection flow below to fill them in.

### If `.env.claude` does NOT exist — Auto-Detect and Ask:

#### 1.1 Detect `AZURE_DEVOPS_ORG` and `AZURE_DEVOPS_PROJECT` from git remote

```bash
git remote -v
```

Parse Azure DevOps remote URLs. Supported formats:
- `https://org@dev.azure.com/org/project/_git/repo` → org=`org`, project=`project`, repo=`repo`
- `https://dev.azure.com/org/project/_git/repo` → org=`org`, project=`project`, repo=`repo`
- `org@vs-ssh.visualstudio.com:v3/org/project/repo` → org=`org`, project=`project`, repo=`repo`

**Extract all three values**: org, project, AND repo name. The repo name is needed for code wiki creation (each repo in a project gets its own wiki).

If detected, present to the user for confirmation:
> "Detected Azure DevOps org: `[org]`, project: `[project]`, repo: `[repo]`. Is this correct? (y/n)"

If not detected or user says no, ask:
> "Enter your Azure DevOps organization name:"
> "Enter your Azure DevOps project name:"

#### 1.2 Detect `TECH_STACK` from project files

Check for these files in order:
- `package.json` exists AND contains `"next"` in dependencies → `nextjs`
- `*.csproj` or `*.sln` exists → `dotnet`
- `pyproject.toml` or `requirements.txt` or `setup.py` exists → `python`

If detected, present to the user:
> "Detected tech stack: `nextjs` (found Next.js in package.json). Is this correct? (y/n)"

If not detected or user says no, ask:
> "What is your tech stack? Options: `nextjs` | `dotnet` | `python`"

#### 1.3 Ask for values that cannot be auto-detected

Ask the user for each of these (use `AskUserQuestion` when available):

1. **`AZURE_DEVOPS_PAT`** (required):
   > "Enter your Azure DevOps Personal Access Token (PAT). Needs permissions: Work Items Read/Write, Code Read/Write, Build Read/Execute."

2. **`DEPLOY_TARGET`** (required):
   > "Where do you deploy? Options: `hetzner` | `azure` | `vercel`"

3. **`STAGING_URL`** (optional):
   > "Enter your staging environment URL (or press Enter to skip):"

4. **`PRODUCTION_URL`** (optional):
   > "Enter your production environment URL (or press Enter to skip):"

#### 1.4 Write `.env.claude`

Generate the file with all values (detected + user-provided):

```
AZURE_DEVOPS_ORG=<detected or entered>
AZURE_DEVOPS_PROJECT=<detected or entered>
AZURE_DEVOPS_REPO=<detected from git remote>
AZURE_DEVOPS_PAT=<entered by user>
DEPLOY_TARGET=<entered by user>
TECH_STACK=<detected or entered>
STAGING_URL=<entered or empty>
PRODUCTION_URL=<entered or empty>
BRANCH_STRATEGY=gitflow
STAGING_PIPELINE_ID=
PRODUCTION_PIPELINE_ID=
```

Set `BRANCH_STRATEGY` to `gitflow` by default.

#### 1.5 Confirm with user

Display the generated `.env.claude` contents (masking the PAT with `****`) and ask:
> "Here's your configuration. Look correct? (y/n)"

If no, let them edit specific fields. Then load all values into environment variables.

## Step 2: Validate Azure DevOps Connection

```bash
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
az devops project show --project "$AZURE_DEVOPS_PROJECT" --output table
```

If this fails, print the error and suggest checking the PAT token permissions (needs Work Items Read/Write, Code Read/Write, Build Read/Execute).

## Step 2.5: Ensure .env.claude and Plugin Files are Gitignored

`.env.claude` contains secrets (PAT token). Plugin-generated files (`.state.md`, wiki, pipeline) should be tracked, but plugin internal files should not.

Check `.gitignore` and append any missing entries:

```bash
# Secrets
git check-ignore -q .env.claude 2>/dev/null || echo ".env.claude" >> .gitignore

# Plugin state and generated files that should NOT be committed
git check-ignore -q .state.md 2>/dev/null || echo ".state.md" >> .gitignore
```

## Step 2.6: Configure Local Project Settings

Create `.claude/settings.local.json` with the required settings for this project:

```bash
mkdir -p .claude
```

Write `.claude/settings.local.json`:

```json
{
  "defaultMode": "bypassPermissions",
  "outputStyle": "Explanatory"
}
```

If the file already exists, read it and merge the settings (preserve any existing keys, only add/overwrite `defaultMode` and `outputStyle`).

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

For each wiki template file in `${CLAUDE_PLUGIN_ROOT:-.}/templates/wiki/`:
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
| nextjs | hetzner | `${CLAUDE_PLUGIN_ROOT:-.}/templates/pipelines/azure-pipelines-nextjs-hetzner.yml` |
| nextjs | vercel | `${CLAUDE_PLUGIN_ROOT:-.}/templates/pipelines/azure-pipelines-nextjs-vercel.yml` |
| dotnet | azure | `${CLAUDE_PLUGIN_ROOT:-.}/templates/pipelines/azure-pipelines-dotnet-azure.yml` |

1. Copy the matching template to `azure-pipelines.yml` at the repository root
2. If no exact match exists, warn the user and skip pipeline generation:
   > "No pipeline template for TECH_STACK=$TECH_STACK + DEPLOY_TARGET=$DEPLOY_TARGET. You'll need to create azure-pipelines.yml manually."
3. If `azure-pipelines.yml` already exists, ask before overwriting

## Step 6: Update CLAUDE.md

Read the `${CLAUDE_PLUGIN_ROOT:-.}/templates/claude-md-wiki-block.md` template and append it to the project's `CLAUDE.md`.

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
[DONE] CLAUDE.md updated with wiki references and context7 requirement
[DONE] Settings configured (bypassPermissions, Explanatory output)
[DONE] .gitignore updated

Next steps:
1. Run /validate-env to verify everything is correct
2. Populate wiki sections with /wiki auto
3. Push azure-pipelines.yml and configure pipeline in Azure DevOps
4. Start developing with /feature or /quick-fix
```

## Step 8: Initialize State File

Copy `.state.md` template from `${CLAUDE_PLUGIN_ROOT:-.}/templates/.state.md.example` to the repository root. Set all fields to their default idle values. This enables `/next` auto-advance from the first workflow.

## Step 9: Commit

Commit all generated files:

```bash
git add docs/wiki/ azure-pipelines.yml CLAUDE.md .claude/settings.local.json .gitignore
git commit -m "chore: bootstrap project with init-project (wiki, pipeline, settings, CLAUDE.md)"
```
