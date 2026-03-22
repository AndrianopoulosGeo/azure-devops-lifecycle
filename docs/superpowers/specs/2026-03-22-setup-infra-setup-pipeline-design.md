# Setup Infra + Setup Pipeline — Design Spec

> Two new commands to automate the full Azure DevOps CI/CD pipeline setup, eliminating all manual steps discovered during project onboarding.

**Date:** 2026-03-22
**Status:** Approved

---

## Problem

`/init-project` copies a static pipeline YAML template but does nothing to make it actually run. Setting up a working pipeline requires 7+ manual steps: creating service connections, variable groups, environments, approval gates, pipeline definitions, and resource authorizations. This was all discovered during a real onboarding session.

## Solution

Two new commands with a clear boundary:

- **`/setup-infra`** — Creates Azure cloud resources (Resource Group, ACR, App Services, Service Principal). Outputs `.env.infra`.
- **`/setup-pipeline`** — Wires Azure DevOps plumbing (service connections, variable groups, environments, approvals, pipeline YAML, resource authorization). Consumes `.env.infra` + `.env.claude` + `.env`.

## Architecture Decision: Hetzner Self-Hosted Agent

All pipelines use `pool: Hetzner` — a shared Linux dedicated server managed by a separate `manage-server.yaml` pipeline in the `INFRA_PROJECT` Azure DevOps project. Both commands check if the agent is online and auto-trigger the server start if needed.

No Windows agent concerns — Linux-native `script:` steps throughout.

---

## Command 1: `/setup-infra`

**Expert voice:** Cloud Infrastructure Engineer
**Allowed tools:** `[Read, Write, Edit, Bash, Glob, Grep, Agent]`

### Prerequisites
- `.env.claude` exists (from `/init-project`)
- Azure CLI installed and logged in
- `DEPLOY_TARGET=azure` in `.env.claude`

### Phase 0: Ask for INFRA_PROJECT
- Ask the user: "Which Azure DevOps project contains your `manage-server.yaml` pipeline?" (e.g., `Infra Server`)
- Store the answer — it will be written to `.env.infra` in Phase 4

### Phase 1: Validate
- Azure CLI authenticated (`az account show`)
- Hetzner agent pool exists and agent is online. Use REST API (not `az pipelines agent list` — that command doesn't exist):
  ```bash
  # Get pool ID
  POOL_ID=$(az devops invoke --area distributedTask --resource pools --query "value[?name=='Hetzner'].id | [0]" -o tsv)
  # List agents in pool
  az devops invoke --area distributedTask --resource agents --route-parameters poolId=$POOL_ID --query "value[?status=='online']" -o table
  ```
- If agent offline → trigger `manage-server.yaml` in `INFRA_PROJECT` with `action: apply` (`az pipelines run`), poll until online

### Phase 2: Detect or Create Resources
For each resource, check if it exists. If yes, confirm with user. If not, offer to create:

1. **Resource Group** — `az group create`
2. **ACR** — `az acr create` (Basic SKU default, ask user)
3. **App Service Plan** — `az appservice plan create` (Linux, B1 default)
4. **Staging App Service** — `az webapp create` (Docker container)
5. **Production App Service** — `az webapp create` (Docker container)

Propose naming defaults based on project name. Ask user for region preference.

### Phase 3: Create Service Principal
- `az ad sp create-for-rbac` scoped to the resource group
- Grant ACR push/pull: `az role assignment create --role AcrPush`
- Grant App Service deploy: `az role assignment create --role Contributor` on the App Services

### Phase 4: Generate `.env.infra`
```
AZURE_CLIENT_ID=<from SP>
AZURE_CLIENT_SECRET=<from SP>
AZURE_TENANT_ID=<from account>
AZURE_SUBSCRIPTION_ID=<from account>
AZURE_RESOURCE_GROUP=<name>
AZURE_ACR_NAME=<name>
AZURE_ACR_LOGIN_SERVER=<name>.azurecr.io
AZURE_APP_SERVICE_STAGING=<name>
AZURE_APP_SERVICE_PRODUCTION=<name>
INFRA_PROJECT=<Azure DevOps project name for manage-server.yaml>
```

### Phase 5: Gitignore + Commit
- Add `.env.infra` to `.gitignore`
- Commit gitignore update

---

## Command 2: `/setup-pipeline`

**Expert voice:** DevOps Engineer
**Allowed tools:** `[Read, Write, Edit, Bash, Glob, Grep, Agent, Skill]`

### Prerequisites
- `.env.claude` exists (from `/init-project`)
- `.env.infra` exists (from `/setup-infra`)
- `.env` exists with application secrets
- Hetzner agent pool exists

### Phase 1: Agent Validation
- Check Hetzner agent is online via REST API:
  ```bash
  POOL_ID=$(az devops invoke --area distributedTask --resource pools --query "value[?name=='Hetzner'].id | [0]" -o tsv)
  az devops invoke --area distributedTask --resource agents --route-parameters poolId=$POOL_ID --query "value[?status=='online']" -o table
  ```
- If offline → read `INFRA_PROJECT` from `.env.infra`, trigger `manage-server.yaml` with `action: apply` (`az pipelines run`)
- Poll until agent comes online (timeout after 5 minutes)
- Verify Docker is available on the agent

### Phase 2: Read All Configuration
- `.env.claude` → PAT, org, project, tech stack, deploy target
- `.env.infra` → SP credentials, ACR, App Services, resource group, infra project
- `.env` → application secrets for variable group
- Detect: repo name, image name (from Dockerfile/docker-compose or project name)

### Phase 3: Create Service Connections (REST API)
- **ARM connection** — ServicePrincipal scheme with SP creds from `.env.infra`
- **ACR connection** — UsernamePassword scheme (NOT ServicePrincipal — fails on personal MSA accounts). Field mapping:
  - `username` = `AZURE_CLIENT_ID` (the Service Principal app ID)
  - `password` = `AZURE_CLIENT_SECRET` (the Service Principal key)
  - `registry` / `url` = `https://{AZURE_ACR_LOGIN_SERVER}`

Store connection IDs for authorization in Phase 6.

### Phase 4: Create Variable Group + Environments
- Create `{project}-secrets` variable group from **two sources**:
  - **From `.env.infra`** (infrastructure vars needed by pipeline): `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` (secret), `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_RESOURCE_GROUP`, `AZURE_ACR_LOGIN_SERVER`, `AZURE_APP_SERVICE_STAGING`, `AZURE_APP_SERVICE_PRODUCTION`
  - **From `.env`** (application secrets): all application-specific vars (DB strings, API keys, etc.)
  - Non-secret vars created in bulk, secret vars one-at-a-time with `--secret true`
  - Common secrets to detect: any var containing `SECRET`, `KEY`, `PASSWORD`, `TOKEN`, `CONNECTION_STRING` in the name
- Create `staging` environment via REST API
- Create `production` environment via REST API
- **Production approval gate:** Try REST API (`_apis/pipelines/checks/configurations`) first. If it fails, fall back to Playwright browser automation:
  1. Navigate to `https://dev.azure.com/{ORG}/{PROJECT}/_environments/{ENV_ID}?view=checks`
  2. Click "Add check" → "Approvals"
  3. Search for project owner in approvers field
  4. Click "Create"
  5. Verify check appears in the list

### Phase 5: Generate Pipeline YAML
All using `pool: Hetzner` with Linux-native `script:` steps.

Files to generate:
```
azure-pipelines.yml                    — main orchestrator (triggers, stages)
pipelines/variables/common.yml         — shared non-secret variables
pipelines/templates/build.yml          — Docker build + conditional ACR push
pipelines/templates/lint.yml           — linter (via Docker or native)
pipelines/templates/unit-tests.yml     — tests (docker-compose or native)
pipelines/templates/e2e-tests.yml      — E2E tests (docker-compose)
pipelines/templates/deploy.yml         — parameterized App Service deployment
scripts/generate-env.sh                — reconstructs .env from variable group values at pipeline runtime
```

**`scripts/generate-env.sh` details:** This script runs as a pipeline step to create a `.env` file from the variable group secrets that Azure DevOps injects as environment variables. It reads `$(VAR_NAME)` pipeline variables and writes them to a `.env` file that Docker containers can consume.

**Tech stack adaptation:**
- `nextjs` → Node in Docker, `npm test`, Playwright E2E
- `dotnet` → `dotnet` in Docker, `dotnet test`
- `python` → native Python or Docker, `pytest`

**ACR push optimization:** Only push images on deployment branches (`staging`, `main`) — skip on PRs and `develop`.

**`develop` branch pipeline behavior:** The `develop` branch trigger runs lint + unit tests + build (no ACR push, no deploy). This is what Phase 7's verification run will execute.

### Phase 5.5: Commit + Push Pipeline Files
Before creating the pipeline definition, the YAML must exist on the remote:
```bash
git add azure-pipelines.yml pipelines/ scripts/generate-env.sh
git commit -m "ci: add pipeline configuration"
git push origin develop
```

### Phase 6: Create Pipeline + Authorize Resources
- `az pipelines create --name {PipelineName} --repository {RepoName} --repository-type tfsgit --branch develop --yml-path azure-pipelines.yml --skip-first-run true`
- Authorize ALL resources via REST API:
  - Agent pool (Hetzner)
  - ARM service connection
  - ACR service connection
  - Variable group
  - Staging environment
  - Production environment
- **CRITICAL:** Authorize with both `allPipelines` AND specific `pipelines[{id}]` — some resources need both.

### Phase 7: Trigger + Verify
- `az pipelines run --id $PIPELINE_ID --branch develop`
- Monitor run status
- If it fails: read logs, analyze, report to user with fix suggestions
- Update `STAGING_PIPELINE_ID` and `PRODUCTION_PIPELINE_ID` in `.env.claude`

### Phase 8: Update Documentation
- Update `CLAUDE.md` with pipeline references
- Update `docs/wiki/deployment.md` if it exists

---

## Changes to Existing Commands

### `/init-project`
- **Remove Step 5** (Generate Azure Pipeline) entirely
- **Update Step 7** summary: replace `[DONE] Pipeline generated (azure-pipelines.yml)` with "Run `/setup-infra` then `/setup-pipeline` to configure CI/CD"
- **Update Step 9** commit command: remove `azure-pipelines.yml` from `git add` since it's no longer generated here
- Keep everything else unchanged

### `/validate-env`
- **Only when `DEPLOY_TARGET=azure`**, add `.env.infra` checks:
  - File exists
  - Required fields present: `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_RESOURCE_GROUP`, `AZURE_ACR_NAME`, `AZURE_ACR_LOGIN_SERVER`, `AZURE_APP_SERVICE_STAGING`, `AZURE_APP_SERVICE_PRODUCTION`, `INFRA_PROJECT`
  - SP credentials valid (`az login --service-principal` test)
- Add pipeline resource checks (service connections, variable group, environments exist)
- Add Hetzner agent online check (via REST API, not `az pipelines agent list`)

### Pipeline templates cleanup
- Remove `templates/pipelines/azure-pipelines-*.yml` (old static templates)
- `/setup-pipeline` generates project-specific pipelines dynamically

---

## Config File Architecture

| File | Created by | Contains | Gitignored |
|------|-----------|----------|------------|
| `.env.claude` | `/init-project` | DevOps PAT, org/project, tech stack, deploy target, pipeline IDs | Yes |
| `.env.infra` | `/setup-infra` | Service Principal creds, Azure resource names, infra project | Yes |
| `.env` | User (manual) | Application secrets (DB strings, API keys, JWT secrets) | Yes |

No duplication — `AZURE_SUBSCRIPTION_ID` lives in `.env.infra` only (removed from `.env.claude` Azure-specific fields).

**Dependency flow:**
```
.env.claude  ──→  /setup-infra  ──→  .env.infra
                                         │
.env.claude  ──→  /setup-pipeline  ←─────┘
.env         ──→  /setup-pipeline
```

---

## Updated Workflow Chain

```
Before:  /init-project → /feature → /develop → /staging → /release
After:   /init-project → /setup-infra → /setup-pipeline → /feature → /develop → /staging → /release
                          (once)         (once)
```

The new commands run once per project. After that, the existing workflow is unchanged.

---

## Key Gotchas Documented (from real session)

1. **ACR service connection must use UsernamePassword scheme** — ServicePrincipal scheme fails on personal MSA accounts
2. **Every resource needs explicit pipeline authorization** — agent pool, service connections, variable groups, environments
3. **Authorize with both `allPipelines` AND specific pipeline ID** — some resources need both
4. **Secret variables must be added one-at-a-time** — `--secret true` flag, cannot be bulk-created
5. **Environment creation requires REST API** — `az` CLI has no `az devops environment create` command
6. **Production approval check API is preview** — fall back to Playwright if REST API fails
7. **ACR push only on deployment branches** — skip on PRs and develop to save time/cost
8. **Agent may be offline** — auto-trigger `manage-server.yaml` and wait before proceeding
9. **`az pipelines agent list` doesn't exist** — use `az devops invoke` with distributedTask API to check agent status
10. **Pipeline YAML must be pushed before `az pipelines create`** — the definition references a YAML path on the remote, not local disk
11. **Variable group needs infra vars too** — not just `.env` app secrets, also SP creds and resource names from `.env.infra` that the pipeline YAML references
