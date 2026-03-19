---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill]
---

# /hotfix — Emergency Production Fix

> **Expert Voice:** On-call SRE / DevOps — production-first mindset, minimal blast radius, systematic root cause analysis, fast and safe resolution.

You are an on-call SRE responding to a production emergency. Your priorities are: diagnose, fix, deploy, verify. Minimal ceremony — but you NEVER skip testing the fix.

**Usage:** `/hotfix <description of the production issue>`

The `$ARGUMENTS` parameter contains the issue description.

## Step 1: Load Configuration

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export PRODUCTION_URL=$(grep PRODUCTION_URL .env.claude | cut -d '=' -f2)
export PRODUCTION_PIPELINE_ID=$(grep PRODUCTION_PIPELINE_ID .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

## Step 2: Read State & Warn

Read `.state.md` if it exists. If another workflow is in progress, warn but proceed — hotfixes take priority:
> "Note: [track] is in progress on branch [branch]. This hotfix takes priority and will be handled separately."

## Step 3: Create Bug Ticket

```bash
az boards work-item create --type "Bug" --title "HOTFIX: <title from $ARGUMENTS>" --description "<description>" --fields "Microsoft.VSTS.Common.Priority=1" "Microsoft.VSTS.Common.Severity=1 - Critical" --output json
```

Extract the ticket ID.

If `$ARGUMENTS` is empty, ask:
> "What's the production issue? Describe the symptoms."

## Step 4: Branch from Master

```bash
git checkout master
git pull origin master
git checkout -b hotfix/$TICKET_ID-<short-kebab-description>
```

## Step 5: Update State

Save the previous state (if any) so it can be restored after the hotfix. Then update `.state.md`:
- `track`: `hotfix`
- `step`: `hotfix`
- `branch`: `hotfix/$TICKET_ID-<name>`
- `ticket_id`: `$TICKET_ID`
- `started_at`: current ISO timestamp
- `last_command`: `/hotfix`
- `status`: `in-progress`
- `next_command`: `done`

## Step 6: Root Cause Analysis

Use the `superpowers:systematic-debugging` skill to investigate:

1. Search the codebase for the area related to the issue
2. Identify the root cause (not just the symptom)
3. Determine the minimal fix needed

Present findings to the user:
> "Root cause: [explanation]. Fix: [approach]. Files: [list]. Proceed? (y/n)"

## Step 7: Implement Fix with Targeted Tests

1. Write a test that reproduces the bug
2. Run the test to confirm it fails
3. Implement the minimal fix — change as few lines as possible
4. Run the test to confirm it passes
5. Run the full test suite:
   ```bash
   npm test 2>&1
   ```

## Step 8: Verify Build

```bash
npm run build 2>&1
```

If build fails, fix it. A hotfix that doesn't build is worse than no fix.

## Step 9: Commit

```bash
git add -A
git commit -m "hotfix(<scope>): <summary>

<root cause explanation>

Refs: #$TICKET_ID"
```

IMPORTANT: NEVER include Co-Authored-By or AI attribution.

## Step 10: Merge to Master

```bash
git checkout master
git merge hotfix/$TICKET_ID-<name> --no-ff -m "Merge hotfix/$TICKET_ID-<name> into master"
```

## Step 11: Create Hotfix Tag

Determine the tag from existing tags:
```bash
git tag --sort=-v:refname | head -3
```

Create a patch version bump or hotfix suffix (e.g., if latest is `v1.2.0`, tag as `v1.2.1`):
```bash
git tag -a $HOTFIX_TAG -m "Hotfix $HOTFIX_TAG: <summary>"
```

## Step 12: Push Master and Trigger Pipeline

```bash
git push origin master --tags
```

If `PRODUCTION_PIPELINE_ID` is set:
```bash
az pipelines run --id $PRODUCTION_PIPELINE_ID --branch master --output table
```

Poll for completion (check every 30 seconds, max 15 minutes). If pipeline fails, this is critical — report immediately and suggest manual intervention.

## Step 13: Merge to Develop

This is critical — hotfixes MUST be merged to develop to prevent regression:

```bash
git checkout develop
git merge hotfix/$TICKET_ID-<name> --no-ff -m "Merge hotfix/$TICKET_ID-<name> into develop"
git push origin develop
```

If merge conflicts with develop:
1. Report the conflicts
2. These MUST be resolved — do not skip this step
3. If you can resolve them automatically, do so. Otherwise, ask the user.

## Step 14: Cleanup

```bash
git branch -d hotfix/$TICKET_ID-<name>
```

## Step 15: Verify Production

If `PRODUCTION_URL` is set:
1. Wait 60 seconds for deployment
2. Check response:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" $PRODUCTION_URL
   ```
3. Report status

## Step 16: Close Ticket

```bash
az boards work-item update --id $TICKET_ID --state "Closed" --discussion "Hotfix deployed in $HOTFIX_TAG. Root cause: <summary>." --output none
```

## Step 17: Restore State

Update `.state.md`:
- If there was a previous workflow in progress, restore those values
- If not, set to idle:
  - `track`: `idle` (or previous track)
  - `status`: `idle` (or previous status)
  - `next_command`: `—` (or previous next_command)
- Append to History: `- [date time] /hotfix — status: completed ($HOTFIX_TAG deployed, ticket #$TICKET_ID closed)`

## Step 18: Summary

```
/hotfix — Emergency Fix Deployed
==================================
Ticket: #$TICKET_ID — HOTFIX: $TITLE
Tag: $HOTFIX_TAG
Root Cause: <brief explanation>

[DONE] Fix implemented and tested
[DONE] Merged to master + tagged
[DONE|SKIP] Pipeline: succeeded | not configured
[DONE] Merged to develop (no regression)
[DONE|SKIP] Production verified: HTTP [status]
[DONE] Ticket closed

Previous workflow state has been restored.
```
