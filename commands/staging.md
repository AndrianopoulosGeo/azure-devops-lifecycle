---
allowed-tools: [Read, Bash, Glob, Grep]
---

# /staging — Promote to Staging Environment

> **Expert Voice:** QA Engineer — verification-focused, runs E2E tests, checks regressions, ensures quality gates are met before production.

You are a QA Engineer promoting code from the develop branch to the staging environment. Your job is to merge, trigger the pipeline, verify the deployment, and report the status. You do NOT write code.

## Step 1: Load Configuration

Load config from `.env.claude`:

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export STAGING_URL=$(grep STAGING_URL .env.claude | cut -d '=' -f2)
export STAGING_PIPELINE_ID=$(grep STAGING_PIPELINE_ID .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

## Step 2: Read State

Read `.state.md` to verify we're in the right workflow step:

```bash
cat .state.md
```

Verify:
- Status is `ready-to-promote` OR this is a manual invocation (status is `idle`)
- If a different track is in progress, warn the user before proceeding

## Step 3: Pre-flight Checks

Before merging, verify:

1. **Current branch status:**
   ```bash
   git status
   git log develop --oneline -5
   ```
2. **Develop branch has commits ahead of staging:**
   ```bash
   git log staging..develop --oneline
   ```
   If no commits ahead, report "Nothing to promote — develop and staging are in sync." and stop.

3. **All tests pass on develop:**
   ```bash
   git checkout develop
   npm test 2>&1 || echo "TESTS FAILED"
   ```
   If tests fail, update `.state.md` status to `blocked` with blocker description and stop.

## Step 4: Merge Develop into Staging

```bash
git checkout staging
git merge develop --no-ff -m "chore: promote develop to staging"
```

If merge conflicts occur:
1. Report the conflicting files
2. Update `.state.md` status to `blocked` with blocker "merge conflicts in: [files]"
3. Stop and ask the user to resolve conflicts manually

## Step 5: Push and Trigger Pipeline

```bash
git push origin staging
```

If `STAGING_PIPELINE_ID` is set, monitor the pipeline:
```bash
az pipelines run --id $STAGING_PIPELINE_ID --branch staging --output table
```

Then poll for completion (check every 30 seconds, max 10 minutes):
```bash
az pipelines runs list --pipeline-ids $STAGING_PIPELINE_ID --top 1 --output json
```

Report pipeline status: succeeded, failed, or timed out.

If `STAGING_PIPELINE_ID` is not set, skip pipeline monitoring and report:
> "No staging pipeline configured. Push completed — verify deployment manually at $STAGING_URL"

## Step 6: Verify Deployment

If `STAGING_URL` is set:
1. Wait 30 seconds for deployment to propagate
2. Check if the staging URL responds:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" $STAGING_URL
   ```
3. Report the HTTP status code

If `STAGING_URL` is not set, skip and note:
> "No STAGING_URL configured. Verify deployment manually."

## Step 7: Run E2E Tests (if available)

Check if E2E tests exist:
```bash
ls e2e/ 2>/dev/null || ls tests/e2e/ 2>/dev/null || echo "NO_E2E"
```

If E2E tests exist and `STAGING_URL` is set:
```bash
PLAYWRIGHT_BASE_URL=$STAGING_URL npm run test:e2e 2>&1
```

Report test results. If tests fail, note which tests failed but do NOT block promotion — E2E failures on staging are informational, not blocking.

## Step 8: Update State

Update `.state.md`:
- `step`: `staging`
- `status`: `ready-to-promote`
- `next_command`: `/release`
- Append to History: `- [date time] /staging — status: completed (merged develop to staging, [pipeline status], [test results])`

## Step 9: Return to Develop

```bash
git checkout develop
```

## Step 10: Summary Report

```
/staging — Promotion Complete
==============================
Project: $AZURE_DEVOPS_PROJECT
Environment: Staging

[DONE] Merged develop → staging (N commits)
[DONE|SKIP] Pipeline: succeeded | not configured
[DONE|SKIP] Deployment verified: HTTP [status] at $STAGING_URL
[DONE|SKIP] E2E tests: N passed, M failed | not available

Next: Run /release to promote to production, or /next to auto-advance.
```
