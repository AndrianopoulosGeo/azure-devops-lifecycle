# Claude Skills — Portable Development Lifecycle Automation

A portable package of Claude Code commands that can be dropped into any project and self-configure from a single `.env.claude` file.

## Quick Start

1. Copy the `commands/` folder into your project's `.claude/commands/`
2. Create `.env.claude` at your repo root (see `templates/.env.claude.example`)
3. Run `/init-project` to scaffold wiki, pipelines, and CLAUDE.md references
4. Run `/validate-env` to verify everything is set up correctly

## Development Tracks

| Track | Commands | When |
|-------|----------|------|
| **Full Feature** | `/feature` → `/develop` → `/staging` → `/release` | New features |
| **Quick Fix** | `/quick-fix` → `/staging` → `/release` | Small bugs/improvements |
| **Hotfix** | `/hotfix` | Production emergencies |

## Commands

### Bootstrap
- `/init-project` — Scaffold wiki, generate pipelines, configure CLAUDE.md
- `/validate-env` — Health check for repo setup

### Lifecycle
- `/feature` — Brainstorm, plan, create tickets (Business Analyst + Architect)
- `/develop` — 10-phase TDD implementation (Senior Developer)
- `/quick-fix` — Fast track: skip brainstorm, straight to code (Pragmatic Developer)
- `/hotfix` — Branch from master, fix, deploy (On-call SRE)
- `/staging` — Promote develop → staging, verify (QA Engineer)
- `/release` — Promote staging → master, deploy (Release Manager)

### Knowledge
- `/wiki` — Manage project documentation wiki

### Info (Azure DevOps)
- `/board` — View board state
- `/overview` — Standup report
- `/ticket` — Create work item
- `/update-ticket` — Update work item

## Configuration

All commands read from `.env.claude` at the repo root. See `templates/.env.claude.example` for the full schema.

## Requirements

- Claude Code with superpowers plugin enabled
- Azure DevOps project with PAT token
- Git with Gitflow branching (develop, staging, master)
