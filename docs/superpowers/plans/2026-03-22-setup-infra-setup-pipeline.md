# Setup Infra + Setup Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add two new plugin commands (`/setup-infra` and `/setup-pipeline`) and update three existing files to complete the CI/CD automation gap.

**Architecture:** These are Claude Code plugin commands — structured prompt files in `commands/*.md`, not application source code. Each command is a step-by-step prompt with bash snippets and decision logic that Claude follows when invoked. No unit tests — validation is via the plugin's eval framework.

**Tech Stack:** Markdown command files, Azure CLI, Azure DevOps REST API, Playwright (fallback)

**Design Doc:** `docs/superpowers/specs/2026-03-22-setup-infra-setup-pipeline-design.md`

---

## File Structure

| Action | File | Purpose |
|--------|------|---------|
| Create | `commands/setup-infra.md` | `/setup-infra` command — Azure resource provisioning |
| Create | `commands/setup-pipeline.md` | `/setup-pipeline` command — Azure DevOps pipeline plumbing |
| Modify | `commands/init-project.md` | Remove Step 5 (pipeline generation), update Steps 7 and 9 |
| Modify | `commands/validate-env.md` | Add `.env.infra` checks, agent checks, pipeline resource checks |
| Modify | `templates/.env.claude.example` | Remove Azure deploy-target fields (moved to `.env.infra`) |
| Delete | `templates/pipelines/azure-pipelines-nextjs-hetzner.yml` | Replaced by dynamic generation |
| Delete | `templates/pipelines/azure-pipelines-nextjs-vercel.yml` | Replaced by dynamic generation |
| Delete | `templates/pipelines/azure-pipelines-dotnet-azure.yml` | Replaced by dynamic generation |

---

### Task 1: Create `/setup-infra` command

**Files:**
- Create: `commands/setup-infra.md`

- [ ] **Step 1: Create the command file with frontmatter and expert voice**

```markdown
---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent]
---

# /setup-infra — Azure Infrastructure Provisioning

> **Expert Voice:** Cloud Infrastructure Engineer — provisions cloud resources, creates service principals, ensures infrastructure is ready for CI/CD pipelines.

You are a Cloud Infrastructure Engineer setting up Azure cloud resources for a project. Your job is to detect existing resources or create new ones, generate a Service Principal, and output `.env.infra` with all infrastructure credentials.

**Usage:** `/setup-infra`

**Prerequisites:**
- `.env.claude` exists (from `/init-project`)
- Azure CLI installed and logged in (`az account show` succeeds)
- `DEPLOY_TARGET=azure` in `.env.claude`
```

- [ ] **Step 2: Add Phase 0 — Ask for INFRA_PROJECT**

This must come before any agent checks since the infra project name is needed to trigger the server start.

```markdown
## Phase 0: Ask for INFRA_PROJECT

Ask the user which Azure DevOps project contains the `manage-server.yaml` pipeline for the Hetzner build agent:

> "Which Azure DevOps project contains your `manage-server.yaml` pipeline? (e.g., `Infra Server`)"

Store the answer for Phase 4.
```

- [ ] **Step 3: Add Phase 1 — Validate prerequisites**

Include the correct REST API for agent checking (NOT `az pipelines agent list` which doesn't exist).

```markdown
## Phase 1: Validate

### 1.1 Load configuration

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

If `DEPLOY_TARGET` is not `azure`, stop:
> "This command is for Azure deployments. Your DEPLOY_TARGET is '$DEPLOY_TARGET'. No infrastructure setup needed."

### 1.2 Verify Azure CLI is authenticated

```bash
az account show --output table
```

If this fails, stop:
> "Azure CLI is not logged in. Run `az login` first."

### 1.3 Check Hetzner agent is online

```bash
# Get the Hetzner pool ID
POOL_ID=$(az devops invoke --area distributedTask --resource pools --query "value[?name=='Hetzner'].id | [0]" -o tsv)

if [ -z "$POOL_ID" ]; then
  echo "ERROR: No 'Hetzner' agent pool found in Azure DevOps."
  echo "  → Ensure the self-hosted agent is registered in a pool named 'Hetzner'"
  exit 1
fi

# Check for online agents
ONLINE_COUNT=$(az devops invoke --area distributedTask --resource agents --route-parameters poolId=$POOL_ID --query "value[?status=='online'] | length(@)" -o tsv)
```

If `ONLINE_COUNT` is 0:
1. Look up the `manage-server.yaml` pipeline in `INFRA_PROJECT`:
   ```bash
   az pipelines list --project "$INFRA_PROJECT" --query "[?name=='manage-server'].id | [0]" -o tsv
   ```
2. Trigger it with `action: apply`:
   ```bash
   az pipelines run --id $MANAGE_PIPELINE_ID --project "$INFRA_PROJECT" --parameters "action=apply"
   ```
3. Poll every 30 seconds for up to 5 minutes:
   ```bash
   for i in $(seq 1 10); do
     sleep 30
     ONLINE=$(az devops invoke --area distributedTask --resource agents --route-parameters poolId=$POOL_ID --query "value[?status=='online'] | length(@)" -o tsv)
     if [ "$ONLINE" -gt 0 ]; then echo "Agent is online."; break; fi
     echo "Waiting for agent... ($i/10)"
   done
   ```
4. If still offline after 5 minutes, STOP:
   > "Hetzner agent did not come online after 5 minutes. Check the manage-server pipeline run and agent registration."
```

- [ ] **Step 4: Add Phase 2 — Detect or Create Resources**

```markdown
## Phase 2: Detect or Create Resources

Get the Azure subscription ID and name:

```bash
AZURE_SUBSCRIPTION_ID=$(az account show --query "id" -o tsv)
AZURE_SUBSCRIPTION_NAME=$(az account show --query "name" -o tsv)
```

Ask user for region preference:
> "Which Azure region? (e.g., `westeurope`, `northeurope`, `eastus`). Default: `westeurope`"

Derive naming defaults from the project name (lowercase, no spaces):
```bash
PROJECT_SLUG=$(echo "$AZURE_DEVOPS_PROJECT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
DEFAULT_RG="rg-${PROJECT_SLUG}"
DEFAULT_ACR="${PROJECT_SLUG}acr"  # ACR names must be alphanumeric
DEFAULT_PLAN="plan-${PROJECT_SLUG}"
DEFAULT_APP_STAGING="${PROJECT_SLUG}-staging"
DEFAULT_APP_PROD="${PROJECT_SLUG}-production"
```

Present defaults and ask for confirmation:
> "Proposed resource names:
>   Resource Group: `rg-myproject`
>   ACR: `myprojectacr`
>   App Service Plan: `plan-myproject`
>   Staging App: `myproject-staging`
>   Production App: `myproject-production`
>
> Accept these names? (y/n, or provide custom names)"

### 2.1 Resource Group

```bash
az group show --name "$RG_NAME" 2>/dev/null
```

If exists → confirm with user. If not → create:
```bash
az group create --name "$RG_NAME" --location "$REGION" --output table
```

### 2.2 Azure Container Registry

```bash
az acr show --name "$ACR_NAME" 2>/dev/null
```

If exists → confirm. If not → create:
```bash
az acr create --resource-group "$RG_NAME" --name "$ACR_NAME" --sku Basic --output table
```

Get the login server:
```bash
ACR_LOGIN_SERVER=$(az acr show --name "$ACR_NAME" --query "loginServer" -o tsv)
```

### 2.3 App Service Plan

```bash
az appservice plan show --name "$PLAN_NAME" --resource-group "$RG_NAME" 2>/dev/null
```

If not exists:
```bash
az appservice plan create --name "$PLAN_NAME" --resource-group "$RG_NAME" --is-linux --sku B1 --output table
```

### 2.4 Staging App Service

```bash
az webapp show --name "$APP_STAGING" --resource-group "$RG_NAME" 2>/dev/null
```

If not exists:
```bash
az webapp create --name "$APP_STAGING" --resource-group "$RG_NAME" --plan "$PLAN_NAME" --deployment-container-image-name "$ACR_LOGIN_SERVER/app:latest" --output table
```

### 2.5 Production App Service

Same pattern as staging with `$APP_PROD`.

Report what was detected vs created:
```
Phase 2 — Resources:
  [EXISTS] Resource Group: rg-myproject
  [CREATED] ACR: myprojectacr (myprojectacr.azurecr.io)
  [CREATED] App Service Plan: plan-myproject (Linux, B1)
  [CREATED] Staging: myproject-staging
  [CREATED] Production: myproject-production
```
```

- [ ] **Step 5: Add Phase 3 — Create Service Principal**

```markdown
## Phase 3: Create Service Principal

Create a Service Principal scoped to the resource group:

```bash
SP_OUTPUT=$(az ad sp create-for-rbac --name "sp-${PROJECT_SLUG}-cicd" --role Contributor --scopes "/subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/$RG_NAME" --output json)
AZURE_CLIENT_ID=$(echo "$SP_OUTPUT" | jq -r '.appId')
AZURE_CLIENT_SECRET=$(echo "$SP_OUTPUT" | jq -r '.password')
AZURE_TENANT_ID=$(echo "$SP_OUTPUT" | jq -r '.tenant')
```

Grant ACR push (for pipeline) and pull (for App Service deployments):
```bash
ACR_ID=$(az acr show --name "$ACR_NAME" --query "id" -o tsv)
az role assignment create --assignee "$AZURE_CLIENT_ID" --role AcrPush --scope "$ACR_ID" --output none
az role assignment create --assignee "$AZURE_CLIENT_ID" --role AcrPull --scope "$ACR_ID" --output none
```

Report:
```
Phase 3 — Service Principal:
  [CREATED] sp-myproject-cicd
  [GRANTED] Contributor on rg-myproject
  [GRANTED] AcrPush + AcrPull on myprojectacr
```
```

- [ ] **Step 6: Add Phase 4 — Generate `.env.infra`**

```markdown
## Phase 4: Generate `.env.infra`

Write `.env.infra` with all infrastructure values:

```
# =============================================================================
# Infrastructure Configuration — Generated by /setup-infra
# =============================================================================
# WARNING: Contains Service Principal secrets. Never commit this file.
# =============================================================================

# Service Principal (CI/CD identity)
AZURE_CLIENT_ID=<from Phase 3>
AZURE_CLIENT_SECRET=<from Phase 3>
AZURE_TENANT_ID=<from Phase 3>
AZURE_SUBSCRIPTION_ID=<from Phase 1>

# Azure Resources
AZURE_RESOURCE_GROUP=<from Phase 2>
AZURE_ACR_NAME=<from Phase 2>
AZURE_ACR_LOGIN_SERVER=<from Phase 2>
AZURE_APP_SERVICE_STAGING=<from Phase 2>
AZURE_APP_SERVICE_PRODUCTION=<from Phase 2>

# Build Agent Infrastructure
INFRA_PROJECT=<from Phase 0>
```

Display contents (masking `AZURE_CLIENT_SECRET` with `****`) and confirm with user.
```

- [ ] **Step 7: Add Phase 5 — Gitignore + Commit + Summary**

```markdown
## Phase 5: Gitignore + Commit

Ensure `.env.infra` is gitignored:
```bash
git check-ignore -q .env.infra 2>/dev/null || echo ".env.infra" >> .gitignore
git add .gitignore
git commit -m "chore: add .env.infra to gitignore"
```

## Summary

```
/setup-infra completed:
==========================================
Subscription: $AZURE_SUBSCRIPTION_NAME ($AZURE_SUBSCRIPTION_ID)
Resource Group: $RG_NAME ($REGION)

Resources:
  [DONE] ACR: $ACR_NAME ($ACR_LOGIN_SERVER)
  [DONE] App Service Plan: $PLAN_NAME (Linux, B1)
  [DONE] Staging: $APP_STAGING
  [DONE] Production: $APP_PROD
  [DONE] Service Principal: sp-${PROJECT_SLUG}-cicd
  [DONE] .env.infra generated
  [DONE] .gitignore updated

Next step: Run /setup-pipeline to create the CI/CD pipeline
==========================================
```
```

- [ ] **Step 8: Commit the command file**

```bash
git add commands/setup-infra.md
git commit -m "feat: add /setup-infra command for Azure resource provisioning"
```

---

### Task 2: Create `/setup-pipeline` command

**Files:**
- Create: `commands/setup-pipeline.md`

- [ ] **Step 1: Create the command file with frontmatter, expert voice, and prerequisites**

```markdown
---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill]
---

# /setup-pipeline — Azure DevOps Pipeline Setup

> **Expert Voice:** DevOps Engineer — wires CI/CD plumbing, creates service connections, configures pipeline authorization, generates pipeline YAML.

You are a DevOps Engineer setting up the complete Azure Pipelines CI/CD infrastructure. Your job is to create service connections, variable groups, environments, approval gates, generate pipeline YAML, create the pipeline definition, authorize all resources, and verify the first run.

**Usage:** `/setup-pipeline`

**Prerequisites:**
- `.env.claude` exists (from `/init-project`)
- `.env.infra` exists (from `/setup-infra`)
- `.env` exists with application secrets
- Hetzner agent pool exists with at least one registered agent
```

- [ ] **Step 2: Add Phase 1 — Agent Validation**

Same agent check pattern as `/setup-infra` but reads `INFRA_PROJECT` from `.env.infra`:

```markdown
## Phase 1: Agent Validation

### 1.1 Load all configuration

```bash
# From .env.claude
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_REPO=$(grep AZURE_DEVOPS_REPO .env.claude | cut -d '=' -f2)
export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"

# From .env.infra
export AZURE_CLIENT_ID=$(grep AZURE_CLIENT_ID .env.infra | cut -d '=' -f2)
export AZURE_CLIENT_SECRET=$(grep AZURE_CLIENT_SECRET .env.infra | cut -d '=' -f2)
export AZURE_TENANT_ID=$(grep AZURE_TENANT_ID .env.infra | cut -d '=' -f2)
export AZURE_SUBSCRIPTION_ID=$(grep AZURE_SUBSCRIPTION_ID .env.infra | cut -d '=' -f2)
export AZURE_RESOURCE_GROUP=$(grep AZURE_RESOURCE_GROUP .env.infra | cut -d '=' -f2)
export AZURE_ACR_NAME=$(grep AZURE_ACR_NAME .env.infra | cut -d '=' -f2)
export AZURE_ACR_LOGIN_SERVER=$(grep AZURE_ACR_LOGIN_SERVER .env.infra | cut -d '=' -f2)
export AZURE_APP_SERVICE_STAGING=$(grep AZURE_APP_SERVICE_STAGING .env.infra | cut -d '=' -f2)
export AZURE_APP_SERVICE_PRODUCTION=$(grep AZURE_APP_SERVICE_PRODUCTION .env.infra | cut -d '=' -f2)
export INFRA_PROJECT=$(grep INFRA_PROJECT .env.infra | cut -d '=' -f2)
```

Validate required files exist:
- If `.env.infra` is missing: "`.env.infra` not found. Run `/setup-infra` first."
- If `.env` is missing: "`.env` not found. Create a `.env` file with your application secrets (database URLs, API keys, etc.)."

### 1.2 Check Hetzner agent is online

```bash
POOL_ID=$(az devops invoke --area distributedTask --resource pools --query "value[?name=='Hetzner'].id | [0]" -o tsv)
ONLINE_COUNT=$(az devops invoke --area distributedTask --resource agents --route-parameters poolId=$POOL_ID --query "value[?status=='online'] | length(@)" -o tsv)
```

If offline → trigger `manage-server.yaml` in `$INFRA_PROJECT` and poll (same pattern as `/setup-infra` Phase 1.3).

### 1.3 Detect project settings

```bash
# Image name from project name
IMAGE_NAME=$(echo "$AZURE_DEVOPS_REPO" | tr '[:upper:]' '[:lower:]')

# Check for Dockerfile
if [ ! -f Dockerfile ]; then
  echo "WARNING: No Dockerfile found. Pipeline build steps will need adjustment."
fi
```
```

- [ ] **Step 3: Add Phase 2 — Get Azure DevOps Project ID**

```markdown
## Phase 2: Get Azure DevOps Project ID

The project ID is needed for REST API calls throughout the setup:

```bash
PROJECT_ID=$(az devops project show --project "$AZURE_DEVOPS_PROJECT" --query "id" -o tsv)
```

Get the Azure subscription name (needed for ARM service connection):
```bash
AZURE_SUBSCRIPTION_NAME=$(az account show --subscription "$AZURE_SUBSCRIPTION_ID" --query "name" -o tsv)
```
```

- [ ] **Step 4: Add Phase 3 — Create Service Connections via REST API**

This is the most gotcha-heavy phase. Include the exact REST API payloads.

```markdown
## Phase 3: Create Service Connections (REST API)

**CRITICAL:** Service connections MUST be created via REST API, not `az devops service-endpoint create`, because the CLI does not support all required fields.

**CRITICAL:** Write JSON to temporary files. Do NOT use process substitution (`<(echo ...)`) — it does not work on all platforms.

### 3.1 ARM Service Connection

```bash
cat > /tmp/arm-sc.json << 'JSONEOF'
{
  "name": "Azure-ServiceConnection",
  "type": "AzureRM",
  "url": "https://management.azure.com/",
  "authorization": {
    "scheme": "ServicePrincipal",
    "parameters": {
      "tenantid": "$AZURE_TENANT_ID",
      "serviceprincipalid": "$AZURE_CLIENT_ID",
      "authenticationType": "spnKey",
      "serviceprincipalkey": "$AZURE_CLIENT_SECRET"
    }
  },
  "data": {
    "subscriptionId": "$AZURE_SUBSCRIPTION_ID",
    "subscriptionName": "$AZURE_SUBSCRIPTION_NAME",
    "environment": "AzureCloud",
    "scopeLevel": "Subscription",
    "creationMode": "Manual"
  },
  "isShared": false,
  "serviceEndpointProjectReferences": [
    {
      "projectReference": { "id": "$PROJECT_ID" },
      "name": "Azure-ServiceConnection"
    }
  ]
}
JSONEOF
```

Replace variables in the JSON file using `|` as sed delimiter (subscription names may contain `/`):
```bash
sed -i "s|\$AZURE_TENANT_ID|$AZURE_TENANT_ID|g; s|\$AZURE_CLIENT_ID|$AZURE_CLIENT_ID|g; s|\$AZURE_CLIENT_SECRET|$AZURE_CLIENT_SECRET|g; s|\$AZURE_SUBSCRIPTION_ID|$AZURE_SUBSCRIPTION_ID|g; s|\$AZURE_SUBSCRIPTION_NAME|$AZURE_SUBSCRIPTION_NAME|g; s|\$PROJECT_ID|$PROJECT_ID|g" /tmp/arm-sc.json

ARM_SC_ID=$(curl -s -u ":$AZURE_DEVOPS_PAT" \
  -X POST "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/serviceendpoint/endpoints?api-version=7.1" \
  -H "Content-Type: application/json" \
  -d @/tmp/arm-sc.json | grep -oP '"id"\s*:\s*"\K[^"]+' | head -1)

rm /tmp/arm-sc.json
echo "ARM Service Connection ID: $ARM_SC_ID"
```

### 3.2 ACR Docker Registry Service Connection

**CRITICAL:** Use `UsernamePassword` scheme with SP credentials, NOT `ServicePrincipal` scheme. The ServicePrincipal scheme fails on personal MSA accounts.

- `username` = `AZURE_CLIENT_ID` (the Service Principal app ID)
- `password` = `AZURE_CLIENT_SECRET` (the Service Principal key)

```bash
cat > /tmp/acr-sc.json << 'JSONEOF'
{
  "name": "ACR-ServiceConnection",
  "type": "dockerregistry",
  "url": "https://$AZURE_ACR_LOGIN_SERVER",
  "authorization": {
    "scheme": "UsernamePassword",
    "parameters": {
      "registry": "https://$AZURE_ACR_LOGIN_SERVER",
      "username": "$AZURE_CLIENT_ID",
      "password": "$AZURE_CLIENT_SECRET"
    }
  },
  "data": {
    "registrytype": "Others"
  },
  "isShared": false,
  "serviceEndpointProjectReferences": [
    {
      "projectReference": { "id": "$PROJECT_ID" },
      "name": "ACR-ServiceConnection"
    }
  ]
}
JSONEOF
```

Replace variables and create:
```bash
sed -i "s|\$AZURE_ACR_LOGIN_SERVER|$AZURE_ACR_LOGIN_SERVER|g; s/\$AZURE_CLIENT_ID/$AZURE_CLIENT_ID/g; s/\$AZURE_CLIENT_SECRET/$AZURE_CLIENT_SECRET/g; s/\$PROJECT_ID/$PROJECT_ID/g" /tmp/acr-sc.json

ACR_SC_ID=$(curl -s -u ":$AZURE_DEVOPS_PAT" \
  -X POST "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/serviceendpoint/endpoints?api-version=7.1" \
  -H "Content-Type: application/json" \
  -d @/tmp/acr-sc.json | grep -oP '"id"\s*:\s*"\K[^"]+' | head -1)

rm /tmp/acr-sc.json
echo "ACR Service Connection ID: $ACR_SC_ID"
```

Report:
```
Phase 3 — Service Connections:
  [CREATED] Azure-ServiceConnection (ARM, ServicePrincipal) — ID: $ARM_SC_ID
  [CREATED] ACR-ServiceConnection (Docker, UsernamePassword) — ID: $ACR_SC_ID
```
```

- [ ] **Step 5: Add Phase 4 — Create Variable Group + Environments + Approval Gate**

```markdown
## Phase 4: Create Variable Group + Environments

### 4.1 Create Variable Group

Read `.env.infra` and `.env` to build the variable list. Determine which are secrets:

**Secret detection rule:** Any variable whose name contains `SECRET`, `KEY`, `PASSWORD`, `TOKEN`, or `CONNECTION_STRING` (case-insensitive) is treated as a secret.

**From `.env.infra`** — add these infrastructure vars:
- `AZURE_CLIENT_ID` (non-secret)
- `AZURE_CLIENT_SECRET` (secret)
- `AZURE_TENANT_ID` (non-secret)
- `AZURE_SUBSCRIPTION_ID` (non-secret)
- `AZURE_RESOURCE_GROUP` (non-secret)
- `AZURE_ACR_LOGIN_SERVER` (non-secret)
- `AZURE_APP_SERVICE_STAGING` (non-secret)
- `AZURE_APP_SERVICE_PRODUCTION` (non-secret)

**From `.env`** — add all application variables, classifying each as secret or non-secret.

Create the group with non-secret vars:
```bash
NON_SECRET_VARS=""  # Build --variables argument from non-secret vars
# e.g., --variables AZURE_CLIENT_ID=$AZURE_CLIENT_ID AZURE_TENANT_ID=$AZURE_TENANT_ID ...

VG_OUTPUT=$(az pipelines variable-group create \
  --name "${AZURE_DEVOPS_PROJECT}-secrets" \
  --authorize true \
  --variables $NON_SECRET_VARS \
  --output json)
VG_ID=$(echo "$VG_OUTPUT" | grep -oP '"id"\s*:\s*\K\d+' | head -1)
```

Then add secret vars one-at-a-time:
```bash
# For each secret variable:
az pipelines variable-group variable create --group-id $VG_ID --name "AZURE_CLIENT_SECRET" --value "$AZURE_CLIENT_SECRET" --secret true
# Repeat for each secret from .env
```

### 4.2 Create Environments via REST API

**Note:** `az` CLI has no `az devops environment create` command. Must use REST API.

```bash
# Create staging environment
STAGING_ENV_ID=$(curl -s -u ":$AZURE_DEVOPS_PAT" \
  -X POST "https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_apis/distributedtask/environments?api-version=7.1" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"staging\", \"description\": \"Staging environment\"}" \
  | grep -oP '"id"\s*:\s*\K\d+' | head -1)

# Create production environment
PROD_ENV_ID=$(curl -s -u ":$AZURE_DEVOPS_PAT" \
  -X POST "https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_apis/distributedtask/environments?api-version=7.1" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"production\", \"description\": \"Production environment\"}" \
  | grep -oP '"id"\s*:\s*\K\d+' | head -1)
```

### 4.3 Production Approval Gate

**Pre-compute the approver identity** (before constructing the REST payload):
```bash
APPROVER_EMAIL=$(az account show --query user.name -o tsv)
APPROVER_NAME=$(az devops user show --user "$APPROVER_EMAIL" --query displayName -o tsv 2>/dev/null || echo "$APPROVER_EMAIL")
```

**Try REST API first** (preview endpoint — may not work on all Azure DevOps versions):

Write the payload to a temp file (avoid inline JSON with special characters):
```bash
cat > /tmp/approval-check.json << EOF
{
  "type": { "id": "8C6F20A7-A545-4486-9777-F762FAFE0D4D", "name": "Approval" },
  "settings": {
    "approvers": [{ "displayName": "$APPROVER_NAME" }],
    "minRequiredApprovers": 1
  },
  "resource": { "type": "environment", "id": "$PROD_ENV_ID" }
}
EOF

APPROVAL_RESULT=$(curl -s -w "%{http_code}" -u ":$AZURE_DEVOPS_PAT" \
  -X POST "https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_apis/pipelines/checks/configurations?api-version=7.1-preview.1" \
  -H "Content-Type: application/json" \
  -d @/tmp/approval-check.json)
rm /tmp/approval-check.json
```

**If REST API fails (non-2xx response), fall back to Playwright:**

1. Navigate to `https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_environments/$PROD_ENV_ID?view=checks`
2. Click "Add check" button
3. Select "Approvals"
4. Search for the project owner in the approvers field
5. Click "Create"
6. Verify the approval check appears in the checks list

Report:
```
Phase 4 — Variable Group & Environments:
  [CREATED] Variable Group: ${AZURE_DEVOPS_PROJECT}-secrets (ID: $VG_ID) — N vars (M secrets)
  [CREATED] Environment: staging (ID: $STAGING_ENV_ID)
  [CREATED] Environment: production (ID: $PROD_ENV_ID)
  [CREATED] Production approval gate — via REST API | Playwright fallback
```
```

- [ ] **Step 6: Add Phase 5 — Generate Pipeline YAML**

This is the longest phase. Generate 8 files adapted to `TECH_STACK`. All use `pool: Hetzner` (Linux).

```markdown
## Phase 5: Generate Pipeline YAML

Generate pipeline files based on `TECH_STACK`. All pipelines use `pool: Hetzner` with Linux-native `script:` steps.

**Create directories:**
```bash
mkdir -p pipelines/variables pipelines/templates scripts
```

### 5.1 Main Orchestrator — `azure-pipelines.yml`

Generate based on `TECH_STACK`. The orchestrator defines:
- **Triggers:** `develop` (lint + test + build), `staging` (+ ACR push + deploy to staging), `main` (+ ACR push + deploy to production)
- **PR triggers:** Same as develop (lint + test + build, no deploy)
- **Variable group:** `${AZURE_DEVOPS_PROJECT}-secrets`
- **Variable template:** `pipelines/variables/common.yml`
- **Stages:** Build → Lint → Unit Tests → E2E Tests → (conditional) ACR Push → (conditional) Deploy

Key structure:
```yaml
trigger:
  branches:
    include: [develop, staging, main]

pr:
  branches:
    include: [develop]

pool: Hetzner

variables:
  - group: ${AZURE_DEVOPS_PROJECT}-secrets
  - template: pipelines/variables/common.yml

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - template: pipelines/templates/build.yml

  - stage: Lint
    dependsOn: Build
    jobs:
      - job: Lint
        steps:
          - template: pipelines/templates/lint.yml

  - stage: UnitTests
    dependsOn: Lint
    jobs:
      - job: UnitTests
        steps:
          - template: pipelines/templates/unit-tests.yml

  - stage: E2ETests
    dependsOn: UnitTests
    jobs:
      - job: E2ETests
        steps:
          - template: pipelines/templates/e2e-tests.yml

  - stage: PushACR
    dependsOn: E2ETests
    condition: and(succeeded(), or(eq(variables['Build.SourceBranch'], 'refs/heads/staging'), eq(variables['Build.SourceBranch'], 'refs/heads/main')))
    jobs:
      - job: Push
        steps:
          - script: |
              docker login $(acrLoginServer) -u $(AZURE_CLIENT_ID) -p $(AZURE_CLIENT_SECRET)
              docker push $(acrLoginServer)/$(imageRepository):$(tag)
              docker push $(acrLoginServer)/$(imageRepository):latest
            displayName: 'Push to ACR'

  - stage: DeployStaging
    dependsOn: PushACR
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/staging'))
    jobs:
      - deployment: DeployStaging
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - template: pipelines/templates/deploy.yml
                  parameters:
                    environment: staging
                    appServiceName: $(AZURE_APP_SERVICE_STAGING)

  - stage: DeployProduction
    dependsOn: PushACR
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployProduction
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - template: pipelines/templates/deploy.yml
                  parameters:
                    environment: production
                    appServiceName: $(AZURE_APP_SERVICE_PRODUCTION)
```

### 5.2 Common Variables — `pipelines/variables/common.yml`

```yaml
variables:
  imageRepository: '$IMAGE_NAME'
  acrLoginServer: '$AZURE_ACR_LOGIN_SERVER'
  dockerfilePath: 'Dockerfile'
  tag: '$(Build.BuildId)'
```

### 5.3 Build Template — `pipelines/templates/build.yml`

Adapt to `TECH_STACK`:

**nextjs / python / dotnet (all use Docker):**
```yaml
steps:
  - checkout: self

  - script: |
      BUILD_DATE=$(git log -1 --format=%cI)
      docker build \
        --build-arg BUILD_DATE="$BUILD_DATE" \
        --build-arg BUILD_ID="$(Build.BuildId)" \
        -t $(acrLoginServer)/$(imageRepository):$(tag) \
        -t $(acrLoginServer)/$(imageRepository):latest \
        -f $(dockerfilePath) .
    displayName: 'Docker Build'
```

**Note:** Build only — no push. The `PushACR` stage handles pushing as a separate stage (push-only, no rebuild). The Hetzner agent retains the Docker layer cache between stages since it's a persistent self-hosted agent.
```

### 5.4 Lint Template — `pipelines/templates/lint.yml`

Adapt to `TECH_STACK`:
- **nextjs:** `docker run --rm $(acrLoginServer)/$(imageRepository):$(tag) npm run lint`
- **dotnet:** `docker run --rm $(acrLoginServer)/$(imageRepository):$(tag) dotnet format --verify-no-changes`
- **python:** `docker run --rm $(acrLoginServer)/$(imageRepository):$(tag) python -m ruff check .`

### 5.5 Unit Tests — `pipelines/templates/unit-tests.yml`

Adapt to `TECH_STACK`:
- **nextjs:** `docker compose run --rm app npm test`
- **dotnet:** `docker compose run --rm app dotnet test`
- **python:** `docker compose run --rm app pytest`

Include `docker compose down` cleanup in a finally block or always-run step.

### 5.6 E2E Tests — `pipelines/templates/e2e-tests.yml`

Adapt to `TECH_STACK`:
- **nextjs:** `docker compose -f docker-compose.e2e.yml up --abort-on-container-exit`
- **dotnet / python:** Similar docker-compose E2E pattern

### 5.7 Deploy Template — `pipelines/templates/deploy.yml`

Parameterized for staging/production:

```yaml
parameters:
  - name: environment
    type: string
  - name: appServiceName
    type: string

steps:
  - script: |
      az login --service-principal -u $(AZURE_CLIENT_ID) -p $(AZURE_CLIENT_SECRET) --tenant $(AZURE_TENANT_ID)
      az webapp config container set \
        --name ${{ parameters.appServiceName }} \
        --resource-group $(AZURE_RESOURCE_GROUP) \
        --docker-custom-image-name $(acrLoginServer)/$(imageRepository):$(tag) \
        --docker-registry-server-url https://$(acrLoginServer) \
        --docker-registry-server-user $(AZURE_CLIENT_ID) \
        --docker-registry-server-password $(AZURE_CLIENT_SECRET)
      az webapp restart --name ${{ parameters.appServiceName }} --resource-group $(AZURE_RESOURCE_GROUP)
    displayName: 'Deploy to ${{ parameters.environment }}'
```

### 5.8 Env Generator — `scripts/generate-env.sh`

This script reconstructs a `.env` file from pipeline variable group secrets at runtime. Docker containers consume it.

```bash
#!/bin/bash
# Generate .env file from Azure DevOps pipeline variables
# Each variable from the variable group is available as an environment variable

ENV_FILE="${1:-.env}"
echo "# Generated by pipeline — $(date -u +%Y-%m-%dT%H:%M:%SZ)" > "$ENV_FILE"

# Add each variable that the pipeline injects
# The specific variables depend on what's in the variable group
```

The exact variable list depends on what's in `.env` at setup time. Generate this script dynamically based on the actual variable names read from `.env`.

Make it executable:
```bash
chmod +x scripts/generate-env.sh
```
```

- [ ] **Step 7: Add Phase 5.5 — Commit + Push Pipeline Files**

```markdown
## Phase 5.5: Commit + Push Pipeline Files

The pipeline YAML must exist on the remote before creating the pipeline definition:

```bash
git add azure-pipelines.yml pipelines/ scripts/generate-env.sh
git commit -m "ci: add pipeline configuration for $TECH_STACK on Hetzner agent"
git push origin develop
```

If push fails (remote has changes):
```bash
git pull --rebase origin develop && git push origin develop
```
```

- [ ] **Step 8: Add Phase 6 — Create Pipeline + Authorize ALL Resources**

```markdown
## Phase 6: Create Pipeline + Authorize Resources

### 6.1 Create Pipeline Definition

```bash
PIPELINE_NAME=$(echo "$AZURE_DEVOPS_PROJECT" | tr ' ' '-')
PIPELINE_OUTPUT=$(az pipelines create \
  --name "$PIPELINE_NAME" \
  --repository "$AZURE_DEVOPS_REPO" \
  --repository-type tfsgit \
  --branch develop \
  --yml-path azure-pipelines.yml \
  --skip-first-run true \
  --output json)
PIPELINE_ID=$(echo "$PIPELINE_OUTPUT" | grep -oP '"id"\s*:\s*\K\d+' | head -1)
echo "Pipeline created: $PIPELINE_NAME (ID: $PIPELINE_ID)"
```

### 6.2 Authorize ALL Resources

**CRITICAL:** Every resource needs explicit pipeline authorization. Authorize with BOTH `allPipelines` AND specific `pipelines[{id}]` — some resources need both.

**Base URL for authorization:**
```bash
AUTH_BASE="https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_apis/pipelines/pipelinepermissions"
```

**Agent Pool:**
```bash
QUEUE_ID=$(curl -s -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_apis/distributedtask/queues?queueName=Hetzner&api-version=7.1" \
  | grep -oP '"id"\s*:\s*\K\d+' | head -1)

curl -s -u ":$AZURE_DEVOPS_PAT" \
  -X PATCH "$AUTH_BASE/queue/$QUEUE_ID?api-version=7.1-preview.1" \
  -H "Content-Type: application/json" \
  -d "{\"pipelines\":[{\"id\":$PIPELINE_ID,\"authorized\":true}],\"allPipelines\":{\"authorized\":true}}"
```

**Service Connections (ARM + ACR):**
```bash
for SC_ID in $ARM_SC_ID $ACR_SC_ID; do
  curl -s -u ":$AZURE_DEVOPS_PAT" \
    -X PATCH "$AUTH_BASE/endpoint/$SC_ID?api-version=7.1-preview.1" \
    -H "Content-Type: application/json" \
    -d "{\"pipelines\":[{\"id\":$PIPELINE_ID,\"authorized\":true}],\"allPipelines\":{\"authorized\":true}}"
done
```

**Variable Group:**
```bash
curl -s -u ":$AZURE_DEVOPS_PAT" \
  -X PATCH "$AUTH_BASE/variablegroup/$VG_ID?api-version=7.1-preview.1" \
  -H "Content-Type: application/json" \
  -d "{\"pipelines\":[{\"id\":$PIPELINE_ID,\"authorized\":true}],\"allPipelines\":{\"authorized\":true}}"
```

**Environments (staging + production):**
```bash
for ENV_ID in $STAGING_ENV_ID $PROD_ENV_ID; do
  curl -s -u ":$AZURE_DEVOPS_PAT" \
    -X PATCH "$AUTH_BASE/environment/$ENV_ID?api-version=7.1-preview.1" \
    -H "Content-Type: application/json" \
    -d "{\"pipelines\":[{\"id\":$PIPELINE_ID,\"authorized\":true}],\"allPipelines\":{\"authorized\":true}}"
done
```

Report:
```
Phase 6 — Pipeline + Authorization:
  [CREATED] Pipeline: $PIPELINE_NAME (ID: $PIPELINE_ID)
  [AUTHORIZED] Agent pool: Hetzner
  [AUTHORIZED] ARM Service Connection
  [AUTHORIZED] ACR Service Connection
  [AUTHORIZED] Variable Group: ${AZURE_DEVOPS_PROJECT}-secrets
  [AUTHORIZED] Environment: staging
  [AUTHORIZED] Environment: production
```
```

- [ ] **Step 9: Add Phase 7 — Trigger + Verify**

```markdown
## Phase 7: Trigger + Verify

### 7.1 Trigger first run

```bash
RUN_OUTPUT=$(az pipelines run --id $PIPELINE_ID --branch develop --output json)
RUN_ID=$(echo "$RUN_OUTPUT" | grep -oP '"id"\s*:\s*\K\d+' | head -1)
echo "Pipeline run triggered: #$RUN_ID"
echo "View: https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_build/results?buildId=$RUN_ID"
```

### 7.2 Monitor the run

Poll until completion:
```bash
for i in $(seq 1 30); do
  sleep 20
  STATUS=$(az pipelines runs show --id $RUN_ID --query "status" -o tsv)
  RESULT=$(az pipelines runs show --id $RUN_ID --query "result" -o tsv)
  echo "Run #$RUN_ID: status=$STATUS result=$RESULT ($i/30)"
  if [ "$STATUS" = "completed" ]; then break; fi
done
```

### 7.3 Handle result

**If succeeded:**
```
Pipeline verification: PASSED
  Run #$RUN_ID completed successfully on develop branch.
  Stages executed: Build → Lint → Unit Tests → E2E Tests
  (No ACR push or deploy — develop branch)
```

**If failed:**
Read the failure logs:
```bash
az pipelines runs show --id $RUN_ID --output json
```

Present the failure to the user with analysis:
```
Pipeline verification: FAILED
  Run #$RUN_ID failed.
  Failing stage: [stage name]
  Error: [extracted error message]

Common fixes:
  - "Service connection not found" → Re-authorize (Phase 6)
  - "Agent pool not authorized" → Authorize queue (Phase 6)
  - "Dockerfile not found" → Check dockerfilePath in pipelines/variables/common.yml
  - "docker compose not found" → Ensure docker-compose.yml exists in repo
```

Ask user whether to investigate further or proceed.

### 7.4 Update `.env.claude` with pipeline ID

```bash
# Update STAGING_PIPELINE_ID and PRODUCTION_PIPELINE_ID
# (Same pipeline, different branch triggers)
sed -i "s/STAGING_PIPELINE_ID=.*/STAGING_PIPELINE_ID=$PIPELINE_ID/" .env.claude
sed -i "s/PRODUCTION_PIPELINE_ID=.*/PRODUCTION_PIPELINE_ID=$PIPELINE_ID/" .env.claude
```
```

- [ ] **Step 10: Add Phase 8 — Update Documentation + Summary**

```markdown
## Phase 8: Update Documentation

### 8.1 Update CLAUDE.md

If `CLAUDE.md` exists, add pipeline information:
- Pipeline name and ID
- Branch-to-environment mapping (develop → CI only, staging → staging deploy, main → production deploy)
- Service connection names

### 8.2 Update wiki

If `docs/wiki/deployment.md` exists, update it with:
- Pipeline configuration
- Environment URLs
- Deployment flow

## Summary

```
/setup-pipeline completed:
==========================================
Pipeline: $PIPELINE_NAME (ID: $PIPELINE_ID)
View: https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_build?definitionId=$PIPELINE_ID

Service Connections:
  [DONE] Azure-ServiceConnection (ARM)
  [DONE] ACR-ServiceConnection (Docker)

Variable Group:
  [DONE] ${AZURE_DEVOPS_PROJECT}-secrets — N vars (M secrets)

Environments:
  [DONE] staging
  [DONE] production (with approval gate)

Pipeline Files:
  [DONE] azure-pipelines.yml
  [DONE] pipelines/variables/common.yml
  [DONE] pipelines/templates/build.yml
  [DONE] pipelines/templates/lint.yml
  [DONE] pipelines/templates/unit-tests.yml
  [DONE] pipelines/templates/e2e-tests.yml
  [DONE] pipelines/templates/deploy.yml
  [DONE] scripts/generate-env.sh

Authorization:
  [DONE] All 6 resources authorized for pipeline

Verification:
  [DONE] First run on develop — PASSED | FAILED (see above)

Branch → Environment:
  develop  → lint + test + build (CI only)
  staging  → lint + test + build + ACR push + deploy to staging
  main     → lint + test + build + ACR push + deploy to production

Next: Your CI/CD is ready. Use /feature or /quick-fix to start developing.
==========================================
```
```

- [ ] **Step 11: Commit the command file**

```bash
git add commands/setup-pipeline.md
git commit -m "feat: add /setup-pipeline command for Azure DevOps CI/CD setup"
```

---

### Task 3: Update `/init-project` — Remove pipeline generation

**Files:**
- Modify: `commands/init-project.md:195-257`

- [ ] **Step 1: Remove Step 5 (Generate Azure Pipeline)**

Remove lines 195-209 (the entire "Step 5: Generate Azure Pipeline" section including the tech stack table, template path references, and all 3 sub-steps).

- [ ] **Step 2: Update Step 7 summary**

Replace:
```
[DONE] Pipeline generated (azure-pipelines.yml)
```
With:
```
[TODO] Run /setup-infra then /setup-pipeline to configure CI/CD
```

Update the "Next steps" section:
```
Next steps:
1. Run /validate-env to verify everything is correct
2. Populate wiki sections with /wiki auto
3. Run /setup-infra to provision Azure resources
4. Run /setup-pipeline to create the CI/CD pipeline
5. Start developing with /feature or /quick-fix
```

- [ ] **Step 3: Update Step 9 commit command**

Remove `azure-pipelines.yml` from the `git add` command:

Before:
```bash
git add docs/wiki/ azure-pipelines.yml CLAUDE.md .claude/settings.local.json .gitignore
```

After:
```bash
git add docs/wiki/ CLAUDE.md .claude/settings.local.json .gitignore
```

Update commit message:
```bash
git commit -m "chore: bootstrap project with init-project (wiki, settings, CLAUDE.md)"
```

- [ ] **Step 4: Commit changes**

```bash
git add commands/init-project.md
git commit -m "refactor(init-project): remove pipeline generation, defer to /setup-pipeline"
```

---

### Task 4: Update `/validate-env` — Add infrastructure and pipeline checks

**Files:**
- Modify: `commands/validate-env.md`

- [ ] **Step 1: Add `.env.infra` loading after `.env.claude` loading**

After the existing `.env.claude` load block, add:

```markdown
### Load `.env.infra` (if DEPLOY_TARGET=azure)

If `DEPLOY_TARGET` is `azure`, attempt to load `.env.infra`:

```bash
if [ "$DEPLOY_TARGET" = "azure" ] && [ -f .env.infra ]; then
  export AZURE_CLIENT_ID=$(grep AZURE_CLIENT_ID .env.infra | cut -d '=' -f2)
  export AZURE_CLIENT_SECRET=$(grep AZURE_CLIENT_SECRET .env.infra | cut -d '=' -f2)
  export AZURE_TENANT_ID=$(grep AZURE_TENANT_ID .env.infra | cut -d '=' -f2)
  export AZURE_SUBSCRIPTION_ID=$(grep AZURE_SUBSCRIPTION_ID .env.infra | cut -d '=' -f2)
  export AZURE_RESOURCE_GROUP=$(grep AZURE_RESOURCE_GROUP .env.infra | cut -d '=' -f2)
  export AZURE_ACR_NAME=$(grep AZURE_ACR_NAME .env.infra | cut -d '=' -f2)
  export AZURE_ACR_LOGIN_SERVER=$(grep AZURE_ACR_LOGIN_SERVER .env.infra | cut -d '=' -f2)
  export AZURE_APP_SERVICE_STAGING=$(grep AZURE_APP_SERVICE_STAGING .env.infra | cut -d '=' -f2)
  export AZURE_APP_SERVICE_PRODUCTION=$(grep AZURE_APP_SERVICE_PRODUCTION .env.infra | cut -d '=' -f2)
  export INFRA_PROJECT=$(grep INFRA_PROJECT .env.infra | cut -d '=' -f2)
fi
```
```

- [ ] **Step 2: Add Section 10 — Infrastructure Checks (Azure only)**

Add after the existing Section 9 (Plugin Dependency Checks):

```markdown
### 10. Infrastructure Checks (DEPLOY_TARGET=azure only)

Skip this entire section if `DEPLOY_TARGET` is not `azure`.

| Check | Pass | Fail |
|-------|------|------|
| `.env.infra` exists | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_CLIENT_ID` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_CLIENT_SECRET` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_TENANT_ID` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_SUBSCRIPTION_ID` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_RESOURCE_GROUP` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_ACR_NAME` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_ACR_LOGIN_SERVER` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_APP_SERVICE_STAGING` set | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_APP_SERVICE_PRODUCTION` set | [PASS] | [FAIL] → Run /setup-infra |
| `INFRA_PROJECT` set | [PASS] | [FAIL] → Run /setup-infra |
| SP credentials valid | `az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID 2>&1` | [FAIL] → SP may be expired, run /setup-infra |

### 11. Hetzner Agent Check (runs for ALL deploy targets, not just azure)

```bash
# NOTE: az pipelines agent list does NOT exist. Use az devops invoke with distributedTask API.
POOL_ID=$(az devops invoke --area distributedTask --resource pools --query "value[?name=='Hetzner'].id | [0]" -o tsv 2>/dev/null)
```

| Check | Pass | Fail |
|-------|------|------|
| Hetzner pool exists | [PASS] Pool ID: $POOL_ID | [FAIL] → Register agent in 'Hetzner' pool |
| Agent online | [PASS] | [WARN] → Agent offline. Start with manage-server.yaml in $INFRA_PROJECT |

### 12. Pipeline Resource Checks (DEPLOY_TARGET=azure only)

Skip if `STAGING_PIPELINE_ID` is not set (pipeline not yet created).

| Check | Pass | Fail |
|-------|------|------|
| Service connection: Azure-ServiceConnection | Query endpoint by name | [FAIL] → Run /setup-pipeline |
| Service connection: ACR-ServiceConnection | Query endpoint by name | [FAIL] → Run /setup-pipeline |
| Variable group: ${AZURE_DEVOPS_PROJECT}-secrets | `az pipelines variable-group list --query "[?name=='...']"` | [FAIL] → Run /setup-pipeline |
| Environment: staging | REST API query | [FAIL] → Run /setup-pipeline |
| Environment: production | REST API query | [FAIL] → Run /setup-pipeline |
```

- [ ] **Step 3: Update the .gitignore check to include `.env.infra`**

Add to section 7:
```markdown
| `.env.infra` is gitignored | `grep -q '.env.infra' .gitignore` | [WARN] → Add `.env.infra` to .gitignore |
```

- [ ] **Step 4: Update the Pipeline Checks section**

Update Section 5 (Pipeline Checks) to reference `/setup-pipeline` instead of `/init-project`:

Before: `[FAIL] → Run /init-project`
After: `[FAIL] → Run /setup-pipeline`

- [ ] **Step 5: Update the example output to include new sections**

Add infrastructure and agent sections to the example output block.

- [ ] **Step 6: Commit changes**

```bash
git add commands/validate-env.md
git commit -m "feat(validate-env): add infrastructure, agent, and pipeline resource checks"
```

---

### Task 5: Update `.env.claude` template + cleanup old pipeline templates

**Files:**
- Modify: `templates/.env.claude.example:52-57`
- Delete: `templates/pipelines/azure-pipelines-nextjs-hetzner.yml`
- Delete: `templates/pipelines/azure-pipelines-nextjs-vercel.yml`
- Delete: `templates/pipelines/azure-pipelines-dotnet-azure.yml`

- [ ] **Step 1: Remove Azure deploy-target fields from `.env.claude` template**

These fields move to `.env.infra` (generated by `/setup-infra`). Replace lines 52-57:

Before:
```
# Optional: Azure deployment (for azure deploy target)
# AZURE_SUBSCRIPTION_ID=
# AZURE_RESOURCE_GROUP=
# AZURE_APP_SERVICE_NAME=
# AZURE_CONTAINER_REGISTRY=
```

After:
```
# Optional: Azure deployment (for azure deploy target)
# Azure infrastructure is managed by /setup-infra → .env.infra
# Run /setup-infra to provision resources and generate .env.infra
```

- [ ] **Step 2: Delete old pipeline templates**

```bash
rm templates/pipelines/azure-pipelines-nextjs-hetzner.yml
rm templates/pipelines/azure-pipelines-nextjs-vercel.yml
rm templates/pipelines/azure-pipelines-dotnet-azure.yml
```

Check if the `templates/pipelines/` directory is now empty. If so, remove it:
```bash
rmdir templates/pipelines/ 2>/dev/null || true
```

- [ ] **Step 3: Commit changes**

```bash
git add templates/.env.claude.example
git rm templates/pipelines/azure-pipelines-nextjs-hetzner.yml
git rm templates/pipelines/azure-pipelines-nextjs-vercel.yml
git rm templates/pipelines/azure-pipelines-dotnet-azure.yml
git commit -m "refactor: move Azure config to .env.infra, remove static pipeline templates"
```

---

### Task 6: Update CLAUDE.md and bump version

**Files:**
- Modify: `CLAUDE.md`
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Update CLAUDE.md**

Add `/setup-infra` and `/setup-pipeline` to the command dependency chain table:

```markdown
| `/setup-infra` | (standalone — Azure CLI only) |
| `/setup-pipeline` | `/setup-infra` for `.env.infra`, Playwright (fallback for approval gate) |
```

Update the Workflow Tracks section to show the setup commands:

```markdown
### Project Setup (one-time)
```
/init-project → /setup-infra → /setup-pipeline
```
```

Update the Configuration section to mention `.env.infra`:

```markdown
### Configuration

All commands read from `.env.claude` (never hardcoded values). Required fields: `AZURE_DEVOPS_ORG`, `AZURE_DEVOPS_PROJECT`, `AZURE_DEVOPS_PAT`, `DEPLOY_TARGET`, `TECH_STACK`.

For Azure deployments, `/setup-infra` generates `.env.infra` with Service Principal credentials and resource names. `/setup-pipeline` reads both files.
```

- [ ] **Step 2: Bump version**

Update `.claude-plugin/plugin.json` version from `1.5.0` to `1.6.0`.

- [ ] **Step 3: Commit changes**

```bash
git add CLAUDE.md .claude-plugin/plugin.json
git commit -m "docs: update CLAUDE.md for setup-infra/setup-pipeline, bump to 1.6.0"
```

---

## Execution Order

Tasks 1-2 are independent (new files). Tasks 3-5 are independent modifications. Task 6 depends on all others being done.

```
Task 1 (setup-infra.md)     ──┐
Task 2 (setup-pipeline.md)  ──┤
Task 3 (init-project.md)    ──┼──→ Task 6 (CLAUDE.md + version)
Task 4 (validate-env.md)    ──┤
Task 5 (templates cleanup)  ──┘
```

Tasks 1-5 can be parallelized. Task 6 must run last.
