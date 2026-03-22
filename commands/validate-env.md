---
allowed-tools: [Read, Bash, Glob, Grep]
---

# /validate-env — Environment Health Check

> **Expert Voice:** DevOps Auditor — methodical, checklist-driven, reports pass/fail with remediation.

You are a DevOps Auditor running a comprehensive health check on the project setup. Check every aspect of the configuration, report results clearly, and provide actionable remediation for any failures.

## Load Configuration

First, attempt to load `.env.claude`:

```bash
if [ ! -f .env.claude ]; then
  echo "[FAIL] .env.claude not found"
  echo "  → Create from template: cp ${CLAUDE_PLUGIN_ROOT:-.}/templates/.env.claude.example .env.claude"
  exit 1
fi
```

Then load all fields:

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_REPO=$(grep AZURE_DEVOPS_REPO .env.claude | cut -d '=' -f2)
export DEPLOY_TARGET=$(grep DEPLOY_TARGET .env.claude | cut -d '=' -f2)
export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
export STAGING_URL=$(grep STAGING_URL .env.claude | cut -d '=' -f2)
export PRODUCTION_URL=$(grep PRODUCTION_URL .env.claude | cut -d '=' -f2)
export BRANCH_STRATEGY=$(grep BRANCH_STRATEGY .env.claude | cut -d '=' -f2)
export STAGING_PIPELINE_ID=$(grep STAGING_PIPELINE_ID .env.claude | cut -d '=' -f2)
export PRODUCTION_PIPELINE_ID=$(grep PRODUCTION_PIPELINE_ID .env.claude | cut -d '=' -f2)
```

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

## Checks

Run ALL checks, even if earlier ones fail. Collect all results and present them together at the end.

### 1. Configuration File Checks

| Check | Command | Pass | Fail |
|-------|---------|------|------|
| `.env.claude` exists | `test -f .env.claude` | [PASS] | [FAIL] → Create from template |
| `AZURE_DEVOPS_ORG` set | `test -n "$AZURE_DEVOPS_ORG"` | [PASS] | [FAIL] → Add to .env.claude |
| `AZURE_DEVOPS_PROJECT` set | `test -n "$AZURE_DEVOPS_PROJECT"` | [PASS] | [FAIL] → Add to .env.claude |
| `AZURE_DEVOPS_REPO` set | `test -n "$AZURE_DEVOPS_REPO"` | [PASS] | [FAIL] → Add to .env.claude (repo name for code wiki) |
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
| `azure-pipelines.yml` exists | [PASS] | [FAIL] → Run /setup-pipeline |
| Pipeline matches DEPLOY_TARGET | [PASS] | [WARN] → Pipeline may be stale, regenerate with /setup-pipeline |

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
| `.env.infra` is gitignored | `grep -q '.env.infra' .gitignore` | [WARN] → Add `.env.infra` to .gitignore |

### 8. State File Check

| Check | Pass | Fail |
|-------|------|------|
| `.state.md` exists | [PASS] | [WARN] → Run /init-project to create state file |

### 9. Plugin Dependency Checks

| Check | Command | Pass | Fail |
|-------|---------|------|------|
| Azure CLI installed | `az --version 2>/dev/null` | [PASS] | [FAIL] → Install Azure CLI: https://aka.ms/installazurecli |
| DevOps extension | `az extension show --name azure-devops 2>/dev/null` | [PASS] | [FAIL] → Run: `az extension add --name azure-devops` |
| `superpowers` plugin | Check if superpowers skills are loadable | [PASS] | [WARN] → Install: `/plugin install superpowers@claude-plugins-official` |
| `pr-review-toolkit` plugin | Check if pr-review-toolkit skills are loadable | [PASS] | [WARN] → Install: `/plugin install pr-review-toolkit@claude-plugins-official` |

### 10. Infrastructure Checks (DEPLOY_TARGET=azure only)

> Skip this entire section if `DEPLOY_TARGET` is not `azure`.

| Check | Command | Pass | Fail |
|-------|---------|------|------|
| `.env.infra` exists | `test -f .env.infra` | [PASS] | [FAIL] → Run /setup-infra |
| `AZURE_CLIENT_ID` set | `test -n "$AZURE_CLIENT_ID"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_CLIENT_SECRET` set | `test -n "$AZURE_CLIENT_SECRET"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_TENANT_ID` set | `test -n "$AZURE_TENANT_ID"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_SUBSCRIPTION_ID` set | `test -n "$AZURE_SUBSCRIPTION_ID"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_RESOURCE_GROUP` set | `test -n "$AZURE_RESOURCE_GROUP"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_ACR_NAME` set | `test -n "$AZURE_ACR_NAME"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_ACR_LOGIN_SERVER` set | `test -n "$AZURE_ACR_LOGIN_SERVER"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_APP_SERVICE_STAGING` set | `test -n "$AZURE_APP_SERVICE_STAGING"` | [PASS] | [FAIL] → Add to .env.infra |
| `AZURE_APP_SERVICE_PRODUCTION` set | `test -n "$AZURE_APP_SERVICE_PRODUCTION"` | [PASS] | [FAIL] → Add to .env.infra |
| `INFRA_PROJECT` set | `test -n "$INFRA_PROJECT"` | [PASS] | [FAIL] → Add to .env.infra |
| SP credentials valid | `az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID` | [PASS] | [FAIL] → Check SP credentials in .env.infra |

### 11. Hetzner Agent Check

> NOTE: `az pipelines agent list` does NOT exist. Use `az devops invoke` instead.

```bash
# Check if the Hetzner pool exists and agent is online
az devops invoke --area distributedtask --resource pools --route-parameters --output json 2>/dev/null
```

| Check | Pass | Fail |
|-------|------|------|
| Hetzner pool exists | [PASS] | [WARN] → Hetzner agent pool not found in Azure DevOps |
| Agent is online | [PASS] | [WARN] → Hetzner agent is offline, check the self-hosted agent |

### 12. Pipeline Resource Checks (DEPLOY_TARGET=azure only)

> Skip this entire section if `DEPLOY_TARGET` is not `azure` or if `STAGING_PIPELINE_ID` is not set.

| Check | Command | Pass | Fail |
|-------|---------|------|------|
| Service connection: Azure-ServiceConnection | `az devops service-endpoint list --output json` | [PASS] | [FAIL] → Create Azure-ServiceConnection in Azure DevOps |
| Service connection: ACR-ServiceConnection | `az devops service-endpoint list --output json` | [PASS] | [FAIL] → Create ACR-ServiceConnection in Azure DevOps |
| Variable group: ${AZURE_DEVOPS_PROJECT}-secrets | `az pipelines variable-group list --output json` | [PASS] | [FAIL] → Create variable group in Azure DevOps |
| Environment: staging | `az devops invoke --area distributedtask --resource environments --output json` | [PASS] | [FAIL] → Create staging environment in Azure DevOps |
| Environment: production | `az devops invoke --area distributedtask --resource environments --output json` | [PASS] | [FAIL] → Create production environment in Azure DevOps |

## Output Format

Present all results in a clean table:

```
/validate-env — Environment Health Check
==========================================
Project: $AZURE_DEVOPS_PROJECT
Repo:    $AZURE_DEVOPS_REPO
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

State
  [PASS] .state.md exists

Plugin Dependencies
  [PASS] Azure CLI installed (v2.x.x)
  [PASS] DevOps extension installed
  [WARN] superpowers plugin not detected → /plugin install superpowers@claude-plugins-official
  [WARN] pr-review-toolkit plugin not detected → /plugin install pr-review-toolkit@claude-plugins-official

Security
  [PASS] .env.claude is gitignored
  [PASS] .env.infra is gitignored

Infrastructure (DEPLOY_TARGET=azure)
  [PASS] .env.infra exists
  [PASS] All required fields present (10/10)
  [PASS] Service principal credentials valid

Agent
  [PASS] Hetzner pool exists
  [WARN] Agent is offline → Check the self-hosted agent

Pipeline Resources (DEPLOY_TARGET=azure)
  [PASS] Azure-ServiceConnection exists
  [PASS] ACR-ServiceConnection exists
  [PASS] Variable group: MDT dynamics-secrets exists
  [PASS] Environment: staging exists
  [FAIL] Environment: production missing → Create production environment in Azure DevOps

==========================================
Result: 18 PASS | 2 FAIL | 7 WARN

Action Required:
  1. [FAIL] Create staging branch: git branch staging
  2. [WARN] Populate wiki sections: /wiki auto
```

If all checks pass, end with:
```
All checks passed. Environment is ready for development.
```
