# Installation Guide — Claude Skills Portable Lifecycle

## Prerequisites

Before installing, make sure you have:

- **Claude Code** installed and working
- **Superpowers plugin** enabled (`claude plugins install superpowers`)
- **Azure CLI** installed (`az --version` to verify)
- **Azure DevOps** project with a Personal Access Token (PAT)
  - PAT needs permissions: Work Items (Read/Write), Code (Read/Write), Build (Read/Execute)
- **Git** initialized in your project

## Step 1: Clone the Skills Repo

```bash
git clone https://dev.azure.com/pgSquare/Claude%20Skills/_git/Claude%20Skills
```

Or if you already have it:
```bash
cd "Claude Skills" && git pull origin master
```

## Step 2: Copy Commands to Your Project

From your project's root directory:

```bash
# Create the commands directory if it doesn't exist
mkdir -p .claude/commands/_shared

# Copy all commands
cp "/path/to/Claude Skills/commands/"*.md .claude/commands/
cp "/path/to/Claude Skills/commands/_shared/"*.md .claude/commands/_shared/

# Copy the state template
cp "/path/to/Claude Skills/templates/.state.md.example" .
```

### Windows (PowerShell)

```powershell
$skills = "C:\Users\andri\Dev\Claude Skills"

# Create directories
New-Item -ItemType Directory -Force -Path ".claude\commands\_shared"

# Copy commands
Copy-Item "$skills\commands\*.md" ".claude\commands\"
Copy-Item "$skills\commands\_shared\*.md" ".claude\commands\_shared\"

# Copy state template
Copy-Item "$skills\templates\.state.md.example" "."
```

### Quick Copy Script (Bash — works in Git Bash on Windows)

```bash
SKILLS_DIR="/c/Users/andri/Dev/Claude Skills"
mkdir -p .claude/commands/_shared
cp "$SKILLS_DIR/commands/"*.md .claude/commands/
cp "$SKILLS_DIR/commands/_shared/"*.md .claude/commands/_shared/
cp "$SKILLS_DIR/templates/.state.md.example" .
echo "Commands installed. Run: claude then /init-project"
```

## Step 3: Create `.env.claude`

Copy the template and fill in your values:

```bash
cp "/path/to/Claude Skills/templates/.env.claude.example" .env.claude
```

Then edit `.env.claude`:

```env
# REQUIRED — Azure DevOps
AZURE_DEVOPS_ORG=your-org-name
AZURE_DEVOPS_PROJECT=Your Project Name
AZURE_DEVOPS_PAT=your-pat-token-here

# REQUIRED — Deploy target: hetzner | azure | vercel
DEPLOY_TARGET=hetzner

# REQUIRED — Tech stack: nextjs | dotnet | python
TECH_STACK=nextjs

# OPTIONAL — Environment URLs (leave empty if not set up yet)
STAGING_URL=https://staging.yourproject.com
PRODUCTION_URL=https://yourproject.com

# OPTIONAL — Branch strategy (default: gitflow)
BRANCH_STRATEGY=gitflow

# OPTIONAL — Pipeline IDs (auto-populated after pipeline creation)
STAGING_PIPELINE_ID=
PRODUCTION_PIPELINE_ID=
```

### Security: Add to .gitignore

Make sure `.env.claude` is gitignored (it contains your PAT token):

```bash
echo ".env.claude" >> .gitignore
```

If your `.gitignore` already has `.env*`, you're covered.

## Step 4: Run `/init-project`

Start Claude Code in your project and run:

```
/init-project
```

This will:
- Validate your `.env.claude` configuration
- Test the Azure DevOps connection
- Create `develop` and `staging` branches (if missing)
- Scaffold `docs/wiki/` with 9 template sections
- Generate `azure-pipelines.yml` for your deploy target
- Update `CLAUDE.md` with wiki references
- Create `.state.md` for workflow tracking

## Step 5: Verify Installation

```
/validate-env
```

You should see all checks passing:

```
/validate-env — Environment Health Check
==========================================
Project: Your Project Name
Org:     https://dev.azure.com/your-org

Configuration
  [PASS] .env.claude exists
  [PASS] All required fields present
  ...

Result: N PASS | 0 FAIL | M WARN
```

Fix any FAIL items before proceeding.

## You're Ready!

### Start a New Feature
```
/feature I want to add a user dashboard with analytics
```
Then follow the track: `/develop` → `/staging` → `/release`

Or use `/next` to auto-advance between steps.

### Quick Fix Something
```
/quick-fix the footer copyright year is wrong
```

### Emergency Production Fix
```
/hotfix the login page is returning 500 errors
```

### Check Workflow Status
```
/next
```

### Manage Documentation
```
/wiki status
/wiki auto
```

## Command Reference

| Command | What It Does | When to Use |
|---------|-------------|-------------|
| `/init-project` | Bootstrap project setup | Once per project |
| `/validate-env` | Health check | After setup or when things seem wrong |
| `/next` | Auto-advance workflow | Between pipeline steps |
| `/feature` | Plan + design + tickets | New features |
| `/develop` | 10-phase TDD implementation | After `/feature` |
| `/quick-fix` | Fast fix with ticket | Small bugs, typos, tweaks |
| `/hotfix` | Emergency production fix | Production is broken |
| `/staging` | Promote to staging | After `/develop` or `/quick-fix` |
| `/release` | Deploy to production | After `/staging` verification |
| `/wiki` | Manage project wiki | Keep docs current |
| `/board` | View Azure DevOps board | Check work items |
| `/overview` | Standup report | Daily standup |
| `/ticket` | Create work item | Quick ticket creation |
| `/update-ticket` | Update work item | Change ticket status/fields |

## Workflow Diagram

```
Full Feature Track:
  /feature → /develop → /staging → /release
       ↓         ↓          ↓          ↓
   Plan + Design  TDD     QA + E2E   Tag + Deploy
   + Tickets    + Review  + Verify   + Verify

Quick Fix Track:
  /quick-fix → /staging → /release
       ↓           ↓          ↓
   Ticket + Fix   QA       Deploy
   + Tests

Hotfix Track (bypasses staging):
  /hotfix
     ↓
  Branch from master → Fix → Merge to master + develop → Deploy
```

## Files Created Per Project

| File | Purpose | Git? |
|------|---------|------|
| `.env.claude` | Project config (secrets) | NO (gitignored) |
| `.state.md` | Workflow state tracking | YES |
| `docs/wiki/*.md` | Project documentation (9 files) | YES |
| `azure-pipelines.yml` | CI/CD pipeline | YES |
| `CLAUDE.md` | Updated with wiki refs | YES |

## Updating Commands

When the Claude Skills repo is updated:

```bash
cd "/path/to/Claude Skills" && git pull origin master
cd "/path/to/your-project"
cp "/path/to/Claude Skills/commands/"*.md .claude/commands/
cp "/path/to/Claude Skills/commands/_shared/"*.md .claude/commands/_shared/
```

## Troubleshooting

### Commands not showing in Claude Code
- Verify files are in `.claude/commands/` (not `.claude/command/`)
- Restart Claude Code session (commands load at startup)

### Azure DevOps connection fails
- Check PAT token hasn't expired
- Verify org name matches exactly (case-sensitive)
- Ensure PAT has required permissions

### `/next` says "No active workflow"
- `.state.md` status is `idle` — start a workflow with `/feature`, `/quick-fix`, or `/hotfix`

### Pipeline not triggering
- Set `STAGING_PIPELINE_ID` and `PRODUCTION_PIPELINE_ID` in `.env.claude`
- Pipeline must be configured in Azure DevOps to trigger on branch pushes

### Wiki templates not populated
- Run `/wiki auto` to auto-generate content from codebase analysis
