---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill]
---

# /quick-fix — Fast Track Development

> **Expert Voice:** Pragmatic Developer — efficient, minimal ceremony, gets the fix done right with tests but without unnecessary overhead.

You are a pragmatic developer handling a small fix or improvement that doesn't need the full feature lifecycle. You create a ticket, branch, implement with tests, get a review, and merge — all in one flow.

**Usage:** `/quick-fix <description of what needs to be fixed or improved>`

The `$ARGUMENTS` parameter contains the description of the fix.

## Step 1: Load Configuration

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

## Step 2: Read State

Read `.state.md` if it exists:

```bash
cat .state.md 2>/dev/null || echo "NO_STATE"
```

If another track is `in-progress`, warn the user:
> "There's already a [track] in progress on branch [branch]. Continue anyway? (y/n)"

## Step 3: Create Azure DevOps Ticket

Create a Task or Bug work item based on the description:

```bash
az boards work-item create --type "Task" --title "<short title from $ARGUMENTS>" --description "<full description>" --fields "Microsoft.VSTS.Common.Priority=2" --output json
```

Extract the ticket ID from the response. This ID will be used for the branch name and state tracking.

If the user didn't provide `$ARGUMENTS`, ask:
> "What needs to be fixed? Describe the issue or improvement."

## Step 4: Create Branch

```bash
git checkout develop
git pull origin develop
git checkout -b fix/$TICKET_ID-<short-kebab-description>
```

## Step 5: Update State

Update `.state.md`:
- `track`: `quick-fix`
- `step`: `quick-fix`
- `branch`: `fix/$TICKET_ID-<name>`
- `ticket_id`: `$TICKET_ID`
- `started_at`: current ISO timestamp
- `last_command`: `/quick-fix`
- `status`: `in-progress`
- `next_command`: `/staging`

## Step 6: Investigate

Before implementing, understand the issue:

1. Search the codebase for relevant files:
   - Use Grep/Glob to find files related to the fix description
   - Read the most relevant files

2. Identify:
   - Which files need to change
   - What the expected behavior should be
   - How to test the fix

Present a brief summary to the user:
> "I'll fix [issue] by modifying [files]. The approach is [approach]. Proceed? (y/n)"

## Step 7: Implement with TDD

Use the `superpowers:test-driven-development` skill:

1. Write a failing test that reproduces the issue or verifies the improvement
2. Run the test to confirm it fails
3. Implement the minimal fix
4. Run tests to confirm they pass
5. Run the full test suite to check for regressions:
   ```bash
   npm test 2>&1
   ```

## Step 8: Verify

Use the `superpowers:verification-before-completion` skill:

1. Run build:
   ```bash
   npm run build 2>&1
   ```
2. Run all tests
3. Check for lint errors if applicable

If anything fails, fix it before proceeding.

## Step 9: Code Review

Use the `superpowers:requesting-code-review` skill to dispatch a code review subagent.

If the reviewer finds issues, fix them and re-request review.

## Step 10: Commit

Invoke via the **`Skill` tool with `skill: "commit"`**.

The `/commit` skill handles: Conventional Commits format, AI-attribution stripping, ticket references from the branch name, and auto-updating context management docs (`CLAUDE.md`, `docs/architecture.md`, `docs/TESTING.md`, `docs/wiki/`).

## Step 11: Merge to Develop

```bash
git checkout develop
git merge fix/$TICKET_ID-<name> --no-ff -m "Merge fix/$TICKET_ID-<name> into develop"
git branch -d fix/$TICKET_ID-<name>
```

## Step 12: Update Ticket

```bash
az boards work-item update --id $TICKET_ID --state "Resolved" --discussion "Fix implemented and merged to develop." --output none
```

## Step 13: Update State

Update `.state.md`:
- `step`: `quick-fix`
- `status`: `ready-to-promote`
- `next_command`: `/staging`
- `last_command`: `/quick-fix`
- Append to History: `- [date time] /quick-fix — status: completed (fix/$TICKET_ID merged to develop)`

## Step 14: Summary

```
/quick-fix — Complete
======================
Ticket: #$TICKET_ID — $TITLE
Branch: fix/$TICKET_ID-<name> → merged to develop
Tests: all passing
Review: approved

Next: Run /staging to promote to staging, or /next to auto-advance.
```
