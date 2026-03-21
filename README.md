# Azure DevOps Lifecycle

Full development lifecycle automation with Azure DevOps — feature planning, TDD implementation, staging, release, hotfix, and wiki management with Gitflow branching.

## Installation

```bash
# Add the marketplace
/plugin marketplace add AndrianopoulosGeo/claude-marketplace

# Install the plugin
/plugin install azure-devops-lifecycle@andrianopoulos-marketplace
```

## Prerequisites

- **Azure CLI** with DevOps extension (`az extension add --name azure-devops`)
- **superpowers** plugin (`/plugin install superpowers@claude-plugins-official`)
- **pr-review-toolkit** plugin (`/plugin install pr-review-toolkit@claude-plugins-official`)
- Git with Gitflow branching (develop, staging, master)

### Optional (for UI/design features)
- `frontend-design` plugin
- `context7` MCP server (library documentation)

## Quick Start

1. Create `.env.claude` at your repo root (copy from the plugin's template)
2. Fill in your Azure DevOps org, project, and PAT token
3. Run `/azure-devops-lifecycle:init-project` to scaffold wiki, pipelines, and CLAUDE.md
4. Run `/azure-devops-lifecycle:validate-env` to verify everything is set up

## Development Tracks

| Track | Commands | When |
|-------|----------|------|
| **Full Feature** | `feature` → `develop` → `staging` → `release` | New features |
| **Quick Fix** | `quick-fix` → `staging` → `release` | Small bugs/improvements |
| **Hotfix** | `hotfix` | Production emergencies |

## Commands

All commands are prefixed with `azure-devops-lifecycle:` when installed as a plugin.

### Bootstrap
| Command | Description |
|---------|-------------|
| `/azure-devops-lifecycle:init-project` | Scaffold wiki, generate pipelines, configure CLAUDE.md |
| `/azure-devops-lifecycle:validate-env` | Health check for repo setup and plugin dependencies |

### Lifecycle
| Command | Description |
|---------|-------------|
| `/azure-devops-lifecycle:feature` | Brainstorm, plan, create tickets (Business Analyst + Architect) |
| `/azure-devops-lifecycle:develop` | 10-phase TDD implementation with quality gates (Senior Developer) |
| `/azure-devops-lifecycle:quick-fix` | Fast track: skip brainstorm, straight to code (Pragmatic Developer) |
| `/azure-devops-lifecycle:hotfix` | Branch from master, fix, deploy (On-call SRE) |
| `/azure-devops-lifecycle:staging` | Promote develop → staging, verify (QA Engineer) |
| `/azure-devops-lifecycle:release` | Promote staging → master, deploy (Release Manager) |

### Knowledge
| Command | Description |
|---------|-------------|
| `/azure-devops-lifecycle:wiki` | Manage project wiki — status, update, auto, reset, review |
| `/azure-devops-lifecycle:publish-wiki` | Publish local wiki to Azure DevOps Wiki |
| `/azure-devops-lifecycle:update-claude-md` | Scan codebase and update CLAUDE.md with project context |

### Commit
| Command | Description |
|---------|-------------|
| `/azure-devops-lifecycle:commit` | Smart commit — no AI attribution, auto-updates context docs |

### Orchestration
| Command | Description |
|---------|-------------|
| `/azure-devops-lifecycle:next` | Auto-advance to next step in current workflow |

## Configuration

All commands read from `.env.claude` at your project root.

### Required Fields
- `AZURE_DEVOPS_ORG` — Your Azure DevOps organization
- `AZURE_DEVOPS_PROJECT` — Project name
- `AZURE_DEVOPS_REPO` — Repository name (auto-detected from git remote, used for per-repo code wikis)
- `AZURE_DEVOPS_PAT` — Personal Access Token
- `DEPLOY_TARGET` — `hetzner` | `azure` | `vercel`
- `TECH_STACK` — `nextjs` | `dotnet` | `python`

### Optional Fields
- `STAGING_URL`, `PRODUCTION_URL` — Environment URLs
- `BRANCH_STRATEGY` — Defaults to `gitflow`
- `STAGING_PIPELINE_ID`, `PRODUCTION_PIPELINE_ID` — Auto-populated by init-project

## License

MIT
