# Azure DevOps Lifecycle — Marketplace Plugin Conversion

## Goal

Convert the Claude Skills project into a Claude Code marketplace-installable plugin (`azure-devops-lifecycle`) with a separate marketplace repo (`claude-marketplace`), and bring `feature.md` and `develop.md` up to parity with the design spec.

## Two-Repo Architecture

### Plugin Repo (`github.com/AndrianopoulosGeo/azure-devops-lifecycle`)

```
azure-devops-lifecycle/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── _shared/
│   │   ├── load-config.md
│   │   └── state-management.md
│   ├── init-project.md
│   ├── validate-env.md
│   ├── feature.md
│   ├── develop.md
│   ├── quick-fix.md
│   ├── hotfix.md
│   ├── staging.md
│   ├── release.md
│   ├── wiki.md
│   └── next.md
├── templates/
│   ├── .env.claude.example
│   ├── .state.md.example
│   ├── claude-md-wiki-block.md
│   ├── wiki/
│   └── pipelines/
├── README.md
└── LICENSE
```

### Marketplace Repo (`github.com/AndrianopoulosGeo/claude-marketplace`)

```
claude-marketplace/
├── .claude-plugin/
│   └── marketplace.json
└── README.md
```

## Plugin Manifest

`.claude-plugin/plugin.json`:
```json
{
  "name": "azure-devops-lifecycle",
  "description": "Full development lifecycle automation with Azure DevOps — feature planning, TDD implementation, staging, release, hotfix, and wiki management with Gitflow branching",
  "version": "1.0.0",
  "author": {
    "name": "AndrianopoulosGeo",
    "email": "andrianopoulos.ge@gmail.com"
  },
  "repository": "https://github.com/AndrianopoulosGeo/azure-devops-lifecycle",
  "license": "MIT",
  "keywords": ["azure-devops", "gitflow", "lifecycle", "ci-cd", "deployment", "tdd"]
}
```

## Marketplace Manifest

`.claude-plugin/marketplace.json`:
```json
{
  "name": "andrianopoulos-marketplace",
  "owner": {
    "name": "AndrianopoulosGeo",
    "email": "andrianopoulos.ge@gmail.com"
  },
  "plugins": [
    {
      "name": "azure-devops-lifecycle",
      "description": "Full development lifecycle automation with Azure DevOps",
      "version": "1.0.0",
      "source": {
        "source": "github",
        "repo": "AndrianopoulosGeo/azure-devops-lifecycle"
      },
      "author": {
        "name": "AndrianopoulosGeo",
        "email": "andrianopoulos.ge@gmail.com"
      }
    }
  ]
}
```

## Installation Flow

```bash
# One-time: add marketplace
/plugin marketplace add AndrianopoulosGeo/claude-marketplace

# Install plugin
/plugin install azure-devops-lifecycle@andrianopoulos-marketplace

# Use in any project
/azure-devops-lifecycle:init-project
/azure-devops-lifecycle:feature <description>
/azure-devops-lifecycle:develop <feature-id>
/azure-devops-lifecycle:quick-fix <description>
/azure-devops-lifecycle:hotfix <description>
/azure-devops-lifecycle:staging
/azure-devops-lifecycle:release
/azure-devops-lifecycle:wiki
/azure-devops-lifecycle:next
/azure-devops-lifecycle:validate-env
```

## Migration Steps

1. Copy `feature.md` and `develop.md` from `MDT dynamics/.claude/commands/` into the plugin repo's `commands/` directory
2. Apply the changes listed below to bring them up to design spec parity
3. Create `.claude-plugin/plugin.json` manifest
4. Create the marketplace repo with `marketplace.json`
5. Push both repos to GitHub

## Changes to Existing Commands

### All Commands: Template Path References

`${CLAUDE_PLUGIN_ROOT}` is an environment variable set automatically by Claude Code's plugin runtime. It resolves to the plugin's cached install location (e.g., `~/.claude/plugins/cache/andrianopoulos-marketplace/azure-devops-lifecycle/1.0.0/`). Commands that reference the plugin's own bundled files must use this variable.

Commands that read `.env.claude` and `.state.md` from the **project root** are unaffected — those files live in the user's project, not the plugin.

Affected commands (template path updates only):
- `init-project.md` — copies templates for wiki, pipelines, .env.claude. Change `templates/` → `${CLAUDE_PLUGIN_ROOT}/templates/`. Add fallback for local dev: `${CLAUDE_PLUGIN_ROOT:-.}/templates/`
- `validate-env.md` — references template path in remediation message. Same change.

### init-project.md — Add .env.claude to .gitignore

Add a step to ensure `.env.claude` is gitignored (it contains `AZURE_DEVOPS_PAT`):
```bash
git check-ignore -q .env.claude 2>/dev/null || echo ".env.claude" >> .gitignore
```

### feature.md — Bring Up to Design Spec Parity

Source: MDT dynamics `feature.md` (443 lines). Changes needed:

1. **Add YAML frontmatter** with `allowed-tools`:
   ```yaml
   ---
   allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill]
   ---
   ```
   Note: MDT file currently has no frontmatter. Adding it shifts all line numbers — reference changes by step name, not line number.

2. **Remove hardcoded board link** (Step 11):
   - Before: `https://dev.azure.com/pgSquare/MDT%20dynamics/_boards/board`
   - After: `https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_boards/board`

3. **Make architectural references tech-stack-agnostic** (Step 3): The file already reads `docs/architecture.md`, `docs/decisions/`, `docs/TESTING.md`, plus config files (`package.json`, `next.config.ts`, etc.). Changes:
   - Mark all config file reads as optional: "Read whatever exists in the project"
   - Add note: "These paths refer to the **target project**, not the plugin. Projects may use different doc structures."
   - Remove Next.js-specific assumptions — let `TECH_STACK` from `.env.claude` guide which config files to look for

4. **Make build/test commands tech-stack-aware** (Steps 6+): Replace hardcoded `npm` commands with tech-stack branching:
   ```
   If TECH_STACK=nextjs: npm run build, npm test, npm run test:e2e
   If TECH_STACK=dotnet: dotnet build, dotnet test
   If TECH_STACK=python: pytest, etc.
   ```
   Alternatively, read build/test commands from the project's `CLAUDE.md` or `package.json` scripts.

5. **Make Context7 mandatory in Step 6.0**: Change from optional/implicit to explicit mandatory requirement with `resolve-library-id` + `query-docs`

6. **Keep Design Toolchain (Step 4.5) as-is**: Already present and well-structured. This is optional per feature type.

7. **Keep state update (Step 12) as-is**: Already correctly sets `next_command: /develop`

### develop.md — Bring Up to Design Spec Parity

Source: MDT dynamics `develop.md` (765 lines). This is the largest change. Additions needed:

1. **Add YAML frontmatter** with `allowed-tools`:
   ```yaml
   ---
   allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill, TaskCreate, TaskUpdate, TaskList]
   ---
   ```

2. **Add Phase 0 — Task Orchestration**:
   - Create 10 tasks via `TaskCreate` before any work begins
   - Task list: Setup, Plan Revalidation, Environment Prep, Implementation, Build & Test, Code Simplification, PR Review, Commit, Close Tickets, Merge

3. **Add Phase 0.5 — Checkpoint Pattern**:
   - Every phase ends with: mark completed → TaskList → mark next in_progress → proceed
   - "Never stop between phases. If the task list shows pending phases, you are not done."

4. **Add worktree isolation to Phase 3**:
   - Phase 3.1: Ensure `.worktrees/` is git-ignored
   - Phase 3.2: Flatten branch name for worktree path to avoid nested dirs on Windows (e.g., `feature/blog-section` → `.worktrees/feature--blog-section`). Create with: `git worktree add ".worktrees/$FLAT_BRANCH" -b "$BRANCH_NAME"`
   - Phase 3.3: Install dependencies in worktree
   - Phase 3.4: Verify clean baseline (build + test)
   - All subsequent phases (4-9) work inside worktree
   - Phase 3.6: Fetch library docs via Context7
   - Phase 3.7: Optional WebSearch for new patterns

5. **Add plan revalidation to Phase 2**:
   - Phase 2.4.1: Codebase verification (file paths, function signatures, import paths, recent changes)
   - Phase 2.4.2: Context7 re-verification for all referenced libraries
   - Phase 2.4.3: WebSearch for best practices (targeted, only for complex patterns)
   - Phase 2.4.4: Azure DevOps ticket alignment check
   - Phase 2.5: Auto-fix discrepancies (Minor: silent fix, Medium: fix + log, Major: stop and ask user)

6. **Make build/test commands tech-stack-aware** (Phase 5 and all test re-runs):
   - Read `TECH_STACK` from `.env.claude`
   - Use appropriate commands: `npm run build`/`dotnet build`/`python -m pytest` etc.
   - Or read build/test commands from the project's `CLAUDE.md`

7. **Add Phase 6 — Code Simplification [MANDATORY]**:
   - Summarize context for handoff (modified files, purpose, complexity areas)
   - Run `Agent` with `subagent_type: "pr-review-toolkit:code-simplifier"`
   - Apply suggestions
   - Re-run build + tests

8. **Add Phase 7 — PR Review [MANDATORY]**:
   - Run `Skill` with `skill: "pr-review-toolkit:review-pr"`
   - Fix HIGH and MEDIUM severity issues
   - Run code simplifier again (second pass)
   - Final build + test run

9. **Update Phase 10 — Merge & Cleanup**:
   - Return to main working directory from worktree
   - Merge feature branch to develop
   - Clean up plan files (`rm -f docs/plans/<feature-name>*.md`)
   - Remove worktree and delete feature branch
   - Add state update: `step: develop`, `status: ready-to-promote`, `next_command: /staging`

10. **Add Workflow Enforcement Rules section**:
   - Task list is source of truth
   - Plan files are implementation guide
   - Phases 6 and 7 are mandatory
   - Never combine phases
   - Build + tests run 3 times minimum
   - Code simplifier runs 2 times
   - Always work in worktree
   - Plan deviations must be documented

11. **Add Error Recovery section**:
    - Never skip a failed phase
    - Ask user for direction (debug, revert, or documented exception)
    - Never proceed past quality gate with known failures

## What Stays Unchanged

These commands are already production-ready in the Claude Skills repo:
- `quick-fix.md` — already has frontmatter, state management, config loading
- `hotfix.md` — already has frontmatter, state management, config loading
- `staging.md` — already has frontmatter, state management, config loading
- `release.md` — already has frontmatter, state management, config loading
- `wiki.md` — already has frontmatter, subcommand parsing
- `next.md` — already has frontmatter, state-driven orchestration
- `init-project.md` — needs only template path update
- `validate-env.md` — needs only template path update
- `_shared/load-config.md` — already correct
- `_shared/state-management.md` — already correct

## Plugin Dependencies

The plugin commands reference these external skills/tools. They are **not bundled** — users need them installed separately:

### Required
- `superpowers` plugin (brainstorming, writing-plans, executing-plans, test-driven-development, systematic-debugging, verification-before-completion, dispatching-parallel-agents, requesting-code-review, finishing-a-development-branch)
- `pr-review-toolkit` plugin (review-pr, code-simplifier)
- Azure CLI (`az`) with DevOps extension

### Optional (for UI/design features)
- `frontend-design` plugin
- `ui-ux-pro-max` skill
- `21st magic` MCP tool (component inspiration)
- `context7` MCP server (library documentation)

## Dependency Validation

The plugin manifest does not support a `dependencies` field. Instead, `/validate-env` will check for required plugins and tools:

- Verify `az` CLI is installed and DevOps extension is available
- Check if `superpowers` plugin is installed (try to detect skills availability)
- Check if `pr-review-toolkit` plugin is installed
- Report missing dependencies with install instructions

This makes `/validate-env` the single entry point for "is my setup complete?"

## Versioning & Updates

Follow semver (`MAJOR.MINOR.PATCH`):
- **PATCH**: Bug fixes in commands, wording improvements
- **MINOR**: New commands, new optional features, new template variants
- **MAJOR**: Breaking changes to `.env.claude` schema, removed commands, changed workflow behavior

To publish an update:
1. Bump version in `.claude-plugin/plugin.json`
2. Update version in marketplace's `marketplace.json`
3. Push both repos
4. Users update with: `/plugin update azure-devops-lifecycle@andrianopoulos-marketplace`

## README Update

The README.md needs to be rewritten for marketplace distribution:
- Installation instructions (marketplace add + plugin install)
- Prerequisites (Azure CLI, superpowers plugin, pr-review-toolkit plugin)
- Quick start with `/azure-devops-lifecycle:init-project`
- Command reference table with namespaced names
- Configuration section (`.env.claude` schema)
- Development tracks overview
