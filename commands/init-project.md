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
