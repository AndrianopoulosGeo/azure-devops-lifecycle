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
