## State Management (source this in every lifecycle command)

### Reading State

At the START of every lifecycle command (`/feature`, `/develop`, `/quick-fix`, `/hotfix`, `/staging`, `/release`):

1. Check if `.state.md` exists at the repository root
2. If it exists, read the current state:
   ```bash
   # Quick state check
   grep -A1 "track" .state.md | tail -1 | awk '{print $NF}'
   grep -A1 "step" .state.md | tail -1 | awk '{print $NF}'
   grep -A1 "status" .state.md | tail -1 | awk '{print $NF}'
   ```
3. If the state shows a different track is in progress, WARN the user:
   > "There's already a [track] in progress on branch [branch] (ticket #[id]). Do you want to continue that work, or start fresh?"
4. If `.state.md` doesn't exist, create it from the template with status `idle`

### Writing State

At the END of every lifecycle command, update `.state.md`:

1. Update the `Current Work` table with current values
2. Update `Status` table — set `status`, clear/set `blockers`, set `next_command`
3. Append a line to the `History` section:
   ```markdown
   - [YYYY-MM-DD HH:MM] `/command-name` — status: result (brief summary)
   ```

### Track → Step → Next Command Mapping

| Track | Step Sequence | Next Command |
|-------|--------------|-------------|
| feature | feature → develop → staging → release → idle | `/develop` → `/staging` → `/release` → done |
| quick-fix | quick-fix → staging → release → idle | `/staging` → `/release` → done |
| hotfix | hotfix → idle | done |

### Status Values

| Status | Meaning | Allowed Next Actions |
|--------|---------|---------------------|
| `idle` | No work in progress | Start any track |
| `in-progress` | Command is currently running | Wait or resume |
| `blocked` | Cannot proceed, needs intervention | Fix blocker, then `/next` |
| `awaiting-review` | Code review or approval needed | Review, then `/next` |
| `ready-to-promote` | Ready for next environment | `/staging` or `/release` |
