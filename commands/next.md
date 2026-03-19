---
allowed-tools: [Read, Bash, Skill]
---

# /next — Auto-Advance to Next Step

> **Expert Voice:** Workflow Orchestrator — reads state, determines next action, hands off to the right command.

You are a workflow orchestrator. Your ONLY job is to read the current project state and invoke the next command in the pipeline. You do NOT implement anything yourself.

## Step 1: Read State

Read `.state.md` from the repository root:

```bash
cat .state.md
```

If `.state.md` does not exist:
> "No active workflow. Start one with `/feature`, `/quick-fix`, or `/hotfix`."
Stop.

## Step 2: Determine Next Action

Parse the state fields:
- `track`: Which pipeline are we in?
- `step`: What was the last completed step?
- `status`: Are we blocked or ready?
- `next_command`: What should run next?

### Decision Matrix

| Status | Action |
|--------|--------|
| `idle` | "No active workflow. Start one with `/feature`, `/quick-fix`, or `/hotfix`." — Stop. |
| `blocked` | "Workflow is blocked: [blockers]. Resolve the issue and run `/next` again." — Stop. |
| `awaiting-review` | "Step [step] is awaiting review. Complete the review and run `/next` again." — Stop. |
| `in-progress` | "Step [step] is still in progress. Let it complete or resume with `/[step]`." — Stop. |
| `ready-to-promote` | Read `next_command` and invoke it. |

### Track Routing

If `next_command` is set and status is `ready-to-promote`:

| next_command | Action |
|-------------|--------|
| `/develop` | Invoke the Skill tool with skill: "develop" |
| `/staging` | Invoke the Skill tool with skill: "staging" |
| `/release` | Invoke the Skill tool with skill: "release" |
| `done` | Report: "Workflow complete! Track: [track], Ticket: #[ticket_id]. State reset to idle." Then update `.state.md` to idle. |

## Step 3: Report

Before invoking the next command, print:
```
/next — Auto-advancing workflow
  Track: [track]
  Completed: [step]
  Next: [next_command]
  Branch: [branch]
  Ticket: #[ticket_id]
```

Then invoke the next command.
