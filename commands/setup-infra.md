---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent]
---

# /setup-infra — Azure Infrastructure Provisioning

> **Expert Voice:** Cloud Infrastructure Engineer — provisions Azure resources, configures service principals, ensures infrastructure is ready for CI/CD pipelines.

You are a Cloud Infrastructure Engineer setting up Azure infrastructure for a project. Your job is to provision all required Azure resources (Resource Group, ACR, App Service Plan, App Services), create a service principal, and generate `.env.infra` so the CI/CD pipeline can deploy.

## Prerequisites

Before starting, verify:
1. `.env.claude` exists with valid configuration
2. Azure CLI (`az`) is installed and authenticated
3. The project's `DEPLOY_TARGET` is `azure`

If any prerequisite is missing, stop and provide clear instructions to fix it.

## Phase 0: Ask for INFRA_PROJECT

Ask the user which Azure DevOps project contains the `manage-server.yaml` pipeline:

> "Which Azure DevOps project contains the `manage-server.yaml` pipeline? (This may be different from your application project.)"

Store this value as `INFRA_PROJECT` — it will be written to `.env.infra` in Phase 4.

## Phase 1: Validate Environment

### 1.1 Load Configuration

Load config from `.env.claude`:

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
```

If `DEPLOY_TARGET` is **not** `azure`, stop immediately:

> "This command is only for Azure deployments. Your DEPLOY_TARGET is `$DEPLOY_TARGET`. Use the appropriate deployment setup for your target."

### 1.2 Verify Azure CLI Authentication

```bash
az account show --output table
```

If this fails, stop and instruct the user to run `az login` first.

### 1.3 Check Hetzner Agent Online

Configure Azure DevOps CLI defaults and check the Hetzner agent pool:

```bash
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$INFRA_PROJECT"
```

**IMPORTANT:** `az pipelines agent list` does NOT exist. You must use `az devops invoke`:

```bash
POOL_ID=$(az devops invoke --area distributedTask --resource pools --query "value[?name=='Hetzner'].id | [0]" -o tsv)
ONLINE_COUNT=$(az devops invoke --area distributedTask --resource agents --route-parameters poolId=$POOL_ID --query "value[?status=='online'] | length(@)" -o tsv)
echo "Online agents: $ONLINE_COUNT"
```

If no agents are online (`ONLINE_COUNT` is 0 or empty):

1. Trigger the `manage-server.yaml` pipeline in `$INFRA_PROJECT`:
   ```bash
   az pipelines run --name "manage-server" --project "$INFRA_PROJECT"
   ```
2. Poll every 30 seconds for up to 5 minutes, checking if an agent comes online:
   ```bash
   for i in $(seq 1 10); do
     sleep 30
     ONLINE_COUNT=$(az devops invoke --area distributedTask --resource agents --route-parameters poolId=$POOL_ID --query "value[?status=='online'] | length(@)" -o tsv)
     if [ "$ONLINE_COUNT" -gt 0 ]; then
       echo "Agent is online."
       break
     fi
     echo "Waiting for agent... ($((i*30))s elapsed)"
   done
   ```
3. If still offline after 5 minutes, warn the user but continue (infrastructure provisioning uses Microsoft-hosted agents anyway).

## Phase 2: Detect or Create Azure Resources

### 2.1 Get Subscription Info

```bash
SUBSCRIPTION_ID=$(az account show --query "id" -o tsv)
SUBSCRIPTION_NAME=$(az account show --query "name" -o tsv)
echo "Subscription: $SUBSCRIPTION_NAME ($SUBSCRIPTION_ID)"
```

### 2.2 Ask for Region

Ask the user for the Azure region:

> "Which Azure region? (default: `westeurope`)"

If user presses Enter or confirms, use `westeurope`.

### 2.3 Derive Naming Defaults

Derive resource names from the Azure DevOps project name by slugifying it (lowercase, hyphens, no special characters):

```bash
SLUG=$(echo "$AZURE_DEVOPS_PROJECT" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
```

Present the naming defaults to the user for confirmation:

> **Proposed resource names:**
> - Resource Group: `rg-$SLUG`
> - Container Registry: `acr${SLUG//[-]/}` (no hyphens — ACR names must be alphanumeric)
> - App Service Plan: `plan-$SLUG`
> - Staging App Service: `app-$SLUG-staging`
> - Production App Service: `app-$SLUG-production`
>
> "Accept these names? (y/n, or provide custom names)"

If the user wants custom names, let them override each one individually.

### 2.4 Determine Runtime from TECH_STACK

Before creating App Services, resolve the runtime based on `TECH_STACK`:

```bash
case "$TECH_STACK" in
  nextjs)  RUNTIME="NODE:20-lts" ;;
  dotnet)  RUNTIME="DOTNETCORE:8.0" ;;
  python)  RUNTIME="PYTHON:3.12" ;;
  *)       echo "ERROR: Unknown TECH_STACK=$TECH_STACK"; exit 1 ;;
esac
echo "Using runtime: $RUNTIME"
```

### 2.5 Create or Detect Each Resource

For each resource, check if it exists first. Report whether it was **detected** (already existed) or **created** (newly provisioned).

**If any creation command below fails, stop immediately and report the error to the user.** Do not proceed to the next resource.

**Resource Group:**
```bash
az group show --name "$RG_NAME" --output none 2>/dev/null && echo "EXISTS" || az group create --name "$RG_NAME" --location "$REGION" --output table
```
If creation fails, stop and report the error to the user.

**Azure Container Registry:**
```bash
az acr show --name "$ACR_NAME" --output none 2>/dev/null && echo "EXISTS" || az acr create --resource-group "$RG_NAME" --name "$ACR_NAME" --sku Basic --output table
```
If creation fails, stop and report the error to the user.

Get the ACR login server:
```bash
ACR_LOGIN_SERVER=$(az acr show --name "$ACR_NAME" --query "loginServer" -o tsv)
```

**App Service Plan:**
```bash
az appservice plan show --name "$PLAN_NAME" --resource-group "$RG_NAME" --output none 2>/dev/null && echo "EXISTS" || az appservice plan create --name "$PLAN_NAME" --resource-group "$RG_NAME" --sku B1 --is-linux --output table
```
If creation fails, stop and report the error to the user.

**Staging App Service:**
```bash
az webapp show --name "$APP_STAGING" --resource-group "$RG_NAME" --output none 2>/dev/null && echo "EXISTS" || az webapp create --name "$APP_STAGING" --resource-group "$RG_NAME" --plan "$PLAN_NAME" --runtime "$RUNTIME" --output table
```
If creation fails, stop and report the error to the user.

**Production App Service:**
```bash
az webapp show --name "$APP_PRODUCTION" --resource-group "$RG_NAME" --output none 2>/dev/null && echo "EXISTS" || az webapp create --name "$APP_PRODUCTION" --resource-group "$RG_NAME" --plan "$PLAN_NAME" --runtime "$RUNTIME" --output table
```
If creation fails, stop and report the error to the user.

### 2.6 Report

Print a summary of what was detected vs created:

```
Resource Status:
  Resource Group ($RG_NAME):         detected | created
  Container Registry ($ACR_NAME):    detected | created
  App Service Plan ($PLAN_NAME):     detected | created
  Staging App ($APP_STAGING):        detected | created
  Production App ($APP_PRODUCTION):  detected | created
```

## Phase 3: Create or Reuse Service Principal

### 3.1 Check for Existing Service Principal

Before creating a new SP, check if one already exists:

```bash
EXISTING_SP_APP_ID=$(az ad sp list --display-name "sp-$SLUG-deploy" --query "[0].appId" -o tsv)
```

**If the SP already exists** (`EXISTING_SP_APP_ID` is not empty), warn the user:

> "A service principal `sp-$SLUG-deploy` already exists (appId: `$EXISTING_SP_APP_ID`). The original client secret cannot be recovered.
>
> Choose one:
> 1. **Enter existing secret** — if you still have it (e.g., from a previous `.env.infra`)
> 2. **Rotate credentials** — create a new secret for the existing SP
> 3. **Recreate SP** — delete and recreate the service principal entirely"

- **Option 1 (enter existing secret):** Prompt for the secret. Set `CLIENT_ID=$EXISTING_SP_APP_ID`, `CLIENT_SECRET=<user input>`, and retrieve `TENANT_ID` with:
  ```bash
  TENANT_ID=$(az ad sp show --id "$EXISTING_SP_APP_ID" --query "appOwnerOrganizationId" -o tsv)
  ```
  Skip to ACR role assignment below.

- **Option 2 (rotate):** Reset the credential:
  ```bash
  SP_OUTPUT=$(az ad sp credential reset --id "$EXISTING_SP_APP_ID" --output json)
  CLIENT_ID=$(echo "$SP_OUTPUT" | jq -r '.appId')
  CLIENT_SECRET=$(echo "$SP_OUTPUT" | jq -r '.password')
  TENANT_ID=$(echo "$SP_OUTPUT" | jq -r '.tenant')
  ```

- **Option 3 (recreate):** Delete and create fresh:
  ```bash
  az ad sp delete --id "$EXISTING_SP_APP_ID"
  ```
  Then proceed to step 3.2 below.

### 3.2 Create New Service Principal (if needed)

Only run this if no SP exists or the user chose Option 3 above:

```bash
SP_OUTPUT=$(az ad sp create-for-rbac --name "sp-$SLUG-deploy" --role Contributor --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME" --output json)
```

**IMPORTANT:** Parse the output with `jq -r`, NOT `grep`. Grep is fragile for JSON parsing.

```bash
CLIENT_ID=$(echo "$SP_OUTPUT" | jq -r '.appId')
CLIENT_SECRET=$(echo "$SP_OUTPUT" | jq -r '.password')
TENANT_ID=$(echo "$SP_OUTPUT" | jq -r '.tenant')
```

### 3.3 Grant ACR Roles

Grant ACR roles to the service principal (idempotent — safe to run even if already assigned):

```bash
ACR_ID=$(az acr show --name "$ACR_NAME" --query "id" -o tsv)
az role assignment create --assignee "$CLIENT_ID" --role AcrPush --scope "$ACR_ID" --output none
az role assignment create --assignee "$CLIENT_ID" --role AcrPull --scope "$ACR_ID" --output none
```

Report what was done:

> "Service principal `sp-$SLUG-deploy` ready with Contributor role on `$RG_NAME` and AcrPush + AcrPull on `$ACR_NAME`."

## Phase 4: Generate `.env.infra`

Write the `.env.infra` file with all provisioned values:

```bash
cat > .env.infra <<EOF
AZURE_CLIENT_ID=$CLIENT_ID
AZURE_CLIENT_SECRET=$CLIENT_SECRET
AZURE_TENANT_ID=$TENANT_ID
AZURE_SUBSCRIPTION_ID=$SUBSCRIPTION_ID
AZURE_RESOURCE_GROUP=$RG_NAME
AZURE_ACR_NAME=$ACR_NAME
AZURE_ACR_LOGIN_SERVER=$ACR_LOGIN_SERVER
AZURE_APP_SERVICE_STAGING=$APP_STAGING
AZURE_APP_SERVICE_PRODUCTION=$APP_PRODUCTION
INFRA_PROJECT=$INFRA_PROJECT
EOF
```

Display the contents with the secret masked:

```
Generated .env.infra:

AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=****
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_SUBSCRIPTION_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_RESOURCE_GROUP=rg-myproject
AZURE_ACR_NAME=acrmyproject
AZURE_ACR_LOGIN_SERVER=acrmyproject.azurecr.io
AZURE_APP_SERVICE_STAGING=app-myproject-staging
AZURE_APP_SERVICE_PRODUCTION=app-myproject-production
INFRA_PROJECT=InfraProject
```

Ask the user to confirm:

> "Does this look correct? (y/n)"

If no, let them edit specific fields and regenerate the file.

## Phase 5: Gitignore, Commit, and Summary

### 5.1 Add `.env.infra` to `.gitignore`

`.env.infra` contains secrets (service principal credentials). Ensure it is gitignored:

```bash
git check-ignore -q .env.infra 2>/dev/null || echo ".env.infra" >> .gitignore
```

### 5.2 Commit

Commit the gitignore update:

```bash
git add .gitignore
git commit -m "chore: add .env.infra to gitignore for Azure infrastructure secrets"
```

### 5.3 Summary Report

Print a summary of everything that was done:

```
/setup-infra completed:

Project:      $AZURE_DEVOPS_PROJECT
Subscription: $SUBSCRIPTION_NAME ($SUBSCRIPTION_ID)
Region:       $REGION

[DONE] Azure CLI authenticated
[DONE] Hetzner agent — online | triggered pipeline
[DONE] Resource Group ($RG_NAME) — detected | created
[DONE] Container Registry ($ACR_NAME) — detected | created
[DONE] App Service Plan ($PLAN_NAME) — detected | created
[DONE] Staging App ($APP_STAGING) — detected | created
[DONE] Production App ($APP_PRODUCTION) — detected | created
[DONE] Service Principal (sp-$SLUG-deploy) — created with Contributor + AcrPush + AcrPull
[DONE] .env.infra generated
[DONE] .gitignore updated

Next step: Run /setup-pipeline to configure CI/CD pipelines for these resources.
```
