# Marketplace Plugin Conversion — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the Claude Skills project into a marketplace-installable plugin and set up the marketplace repo.

**Architecture:** Two GitHub repos — plugin repo (azure-devops-lifecycle) with commands/templates, and marketplace repo (claude-marketplace) with a manifest pointing to the plugin. Commands are migrated from the current project + MDT dynamics, with parity fixes applied.

**Tech Stack:** Claude Code plugin system, Markdown commands, YAML frontmatter, Git

**Design Doc:** `docs/superpowers/specs/2026-03-20-marketplace-plugin-conversion-design.md`

---

### Task 1: Create Plugin Manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

- [ ] **Step 1: Create the `.claude-plugin/` directory**

```bash
mkdir -p .claude-plugin
```

- [ ] **Step 2: Write `plugin.json`**

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

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add plugin manifest for marketplace distribution"
```

---

### Task 2: Copy feature.md and develop.md from MDT Dynamics

**Files:**
- Create: `commands/feature.md` (copy from `c:\Users\andri\Dev\MDT dynamics\.claude\commands\feature.md`)
- Create: `commands/develop.md` (copy from `c:\Users\andri\Dev\MDT dynamics\.claude\commands\develop.md`)

- [ ] **Step 1: Copy both files**

```bash
cp "c:/Users/andri/Dev/MDT dynamics/.claude/commands/feature.md" commands/feature.md
cp "c:/Users/andri/Dev/MDT dynamics/.claude/commands/develop.md" commands/develop.md
```

- [ ] **Step 2: Verify files are present**

```bash
ls -la commands/feature.md commands/develop.md
```
Expected: Both files present, feature.md ~443 lines, develop.md ~765 lines.

- [ ] **Step 3: Commit raw copies (before modifications)**

```bash
git add commands/feature.md commands/develop.md
git commit -m "feat: add feature and develop commands from MDT dynamics"
```

---

### Task 3: Fix feature.md — Frontmatter and Hardcoded References

**Files:**
- Modify: `commands/feature.md`

- [ ] **Step 1: Add YAML frontmatter**

The file currently starts with plain text (no `---` delimiters). Prepend:

```yaml
---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill]
---
```

- [ ] **Step 2: Replace hardcoded board link in Step 11**

Find (in STEP 11: SUMMARY):
```
3. **Board link**: `https://dev.azure.com/pgSquare/MDT%20dynamics/_boards/board`
```

Replace with:
```
3. **Board link**: `https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_boards/board`
```

- [ ] **Step 3: Make Step 3 architectural references tech-stack-agnostic**

In STEP 3, after the table of architectural reference files, add this note:

```markdown
> **Portability note:** These paths refer to the **target project**, not the plugin. Read whatever exists — not all projects have all files. Let `TECH_STACK` from `.env.claude` guide which config files to look for (e.g., `package.json` for nextjs, `.csproj` for dotnet, `pyproject.toml` for python).
```

In section 3.2, replace the hardcoded Next.js config file list:

```markdown
### 3.2 Project Configuration (read for context — read whatever exists)

| File | What it provides |
|------|-----------------|
| `package.json` or `*.csproj` or `pyproject.toml` | Dependencies, scripts, tech stack |
| Framework config (`next.config.ts`, `appsettings.json`, etc.) | Framework-specific configuration |
| `tsconfig.json` | TypeScript configuration (if applicable) |
| Test config (`vitest.config.ts`, `playwright.config.ts`, etc.) | Test runner configuration (if applicable) |
```

- [ ] **Step 4: Make Context7 mandatory in Step 6.0**

In STEP 6 (Research & Create Implementation Plan), find the section about fetching documentation. Ensure it says:

```markdown
### 6.0 Fetch Up-to-Date Documentation (MANDATORY)

**Context7 (mandatory):**
Use `resolve-library-id` + `query-docs` to fetch docs for every library the feature touches. This is NOT optional — plans written without verified library docs risk using deprecated or non-existent APIs.
```

- [ ] **Step 5: Make build/test commands tech-stack-aware in Step 6**

In the Task Structure section of Step 6.1, replace hardcoded `npm` references. Change the example:

```markdown
**Build/test commands vary by TECH_STACK:**
- `nextjs`: `npm run build`, `npm test`, `npm run test:e2e`
- `dotnet`: `dotnet build`, `dotnet test`
- `python`: `python -m pytest`, `python -m pytest e2e/`

Alternatively, read the exact build/test commands from the project's `CLAUDE.md` if they are documented there.
```

- [ ] **Step 6: Commit feature.md changes**

```bash
git add commands/feature.md
git commit -m "fix(feature): make portable — remove hardcoded refs, add frontmatter, tech-stack-aware"
```

---

### Task 4: Fix develop.md — Frontmatter, Phase 0, Phase 0.5

**Files:**
- Modify: `commands/develop.md`

- [ ] **Step 1: Add YAML frontmatter**

The file currently starts with `# Automated Feature Development Workflow`. Prepend:

```yaml
---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill, TaskCreate, TaskUpdate, TaskList]
---
```

- [ ] **Step 2: Add Phase 0 — Task Orchestration**

Insert BEFORE the existing Phase 1 content (after the intro paragraph and before `## PHASE 1`). Add:

```markdown
## PHASE 0: CREATE ORCHESTRATION TASK LIST

**Before doing ANYTHING else**, create the full workflow task list using `TaskCreate`. You MUST complete every task in order.

Create these 10 tasks (no `blockedBy` needed — the checkpoints enforce sequential execution):

| # | Subject | activeForm |
|---|---------|------------|
| 1 | Phase 1: Setup & Context Loading | Loading project context and plan files |
| 2 | Phase 2: Plan Revalidation & Generation | Validating plans, generating implementation plan if missing |
| 3 | Phase 3: Environment Preparation | Preparing branch and environment |
| 4 | Phase 4: Implementation | Implementing feature code from plan |
| 5 | Phase 5: Build & Test | Running builds and tests |
| 6 | Phase 6: Code Simplification [MANDATORY] | Running code simplifier |
| 7 | Phase 7: PR Review [MANDATORY] | Running PR review |
| 8 | Phase 8: Commit | Creating commits |
| 9 | Phase 9: Close Tickets & Update Knowledge | Closing tickets and updating docs |
| 10 | Phase 10: Merge to Develop | Merging feature branch to develop |

**After creating all 10 tasks, call `TaskList` to confirm the full list is visible. Then proceed to Phase 1.**

---

## PHASE 0.5: CHECKPOINT PATTERN

Every phase ends with the same checkpoint pattern. When you see **"CHECKPOINT N"**, execute these steps:

1. Mark Phase N as `completed` via `TaskUpdate`
2. Call `TaskList` to see remaining phases
3. Mark Phase N+1 as `in_progress` via `TaskUpdate`
4. Proceed immediately to the next phase

**Never stop between phases.** If the task list shows pending phases, you are not done.
```

- [ ] **Step 3: Add CHECKPOINT markers to all existing phases**

At the end of each existing phase section (Phase 1, 2, 3, 4, 5, 8, 9, 10), add:

```markdown
### CHECKPOINT N
```

Where N matches the phase number.

- [ ] **Step 4: Commit**

```bash
git add commands/develop.md
git commit -m "feat(develop): add task orchestration and checkpoint pattern (Phases 0, 0.5)"
```

---

### Task 5: Fix develop.md — Worktree Isolation (Phase 3)

**Files:**
- Modify: `commands/develop.md`

- [ ] **Step 1: Rewrite Phase 3 for worktree isolation**

Replace the existing Phase 3 content (which currently just creates a branch and installs deps) with:

```markdown
## PHASE 3: ENVIRONMENT PREPARATION (WORKTREE-BASED)

**All implementation work happens in an isolated git worktree.** Your main working directory stays on `develop` so the IDE is never disrupted.

### 3.1 Ensure worktree directory is git-ignored

```bash
git check-ignore -q .worktrees 2>/dev/null || echo ".worktrees/" >> .gitignore && git add .gitignore && git commit -m "chore: add .worktrees to gitignore"
```

### 3.2 Create feature branch in a worktree

Flatten the branch name for the worktree path (avoids nested dirs on Windows):

```bash
git pull origin develop
BRANCH_NAME="feature/[short-feature-name]"
FLAT_BRANCH=$(echo "$BRANCH_NAME" | tr '/' '--')
WORKTREE_PATH=".worktrees/$FLAT_BRANCH"
git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME"
```

Use a descriptive branch name derived from the feature title (e.g., `feature/blog-section`, `feature/service-detail-pages`).

**Store the worktree path** — all subsequent phases (4 through 9) execute inside this directory:

```bash
cd "$WORKTREE_PATH"
```

### 3.3 Install dependencies in worktree

Run the appropriate install command based on `TECH_STACK`:
- `nextjs`: `npm install`
- `dotnet`: `dotnet restore`
- `python`: `pip install -r requirements.txt` (or equivalent)

### 3.4 Verify clean baseline

Run build and tests to confirm a clean starting point:
- `nextjs`: `npm run build && npm test`
- `dotnet`: `dotnet build && dotnet test`
- `python`: `python -m pytest`

**If tests fail:** Report failures and ask the user whether to proceed or investigate. Do NOT continue with a broken baseline.

### 3.5 Set Feature to "Active"

```bash
az boards work-item update --id [FEATURE_ID] --state "Active" --output none
```

### 3.6 Fetch library documentation (Context7)

Use `resolve-library-id` + `query-docs` to fetch **up-to-date documentation** for every library/framework the implementation plan references.

### 3.7 Quick web research (if plan references new patterns)

If the implementation plan introduces patterns, libraries, or integrations not previously used in the project, do a quick `WebSearch` for current best practices.

### CHECKPOINT 3

**IMPORTANT:** From this point forward (Phases 4-9), ALL file edits, builds, tests, and commits happen inside the worktree at `$WORKTREE_PATH`. The main working directory remains untouched on `develop`.
```

- [ ] **Step 2: Commit**

```bash
git add commands/develop.md
git commit -m "feat(develop): add worktree isolation to Phase 3"
```

---

### Task 6: Fix develop.md — Plan Revalidation (Phase 2)

**Files:**
- Modify: `commands/develop.md`

- [ ] **Step 1: Add revalidation subsections to Phase 2**

After the existing Phase 2 content about plan generation (Scenarios A/B/C), add these subsections before the checkpoint:

```markdown
### 2.4 Autonomous Revalidation (NO user intervention)

**This step runs automatically and fixes issues on its own.** The goal is to verify that EVERYTHING the plan claims is still accurate against the current codebase and current library documentation.

#### 2.4.1 Codebase Verification

Check each claim in the plan against reality:

- **File paths**: Do all referenced files exist? Are the line ranges still accurate?
- **Function signatures**: Do the functions/methods referenced in the plan still have the same signatures?
- **Import paths**: Are all import paths valid in the current codebase?
- **Existing patterns**: Do the patterns the plan follows match what the codebase currently uses?
- **Recent changes**: Have any relevant files been modified since the plan was written? (`git log --since` on referenced files)

#### 2.4.2 Documentation Re-Verification (Context7)

Re-fetch up-to-date documentation for ALL libraries referenced in the plan:

Use `resolve-library-id` + `query-docs` for each library the plan references.

**Cross-check**: Verify that API usage in the plan matches the CURRENT library docs. Flag any:
- Deprecated methods or patterns
- Changed API signatures
- New recommended approaches that supersede what the plan uses

#### 2.4.3 Best Practices Re-Verification (WebSearch)

For complex or newer patterns referenced in the plan, do a targeted `WebSearch`:
- Verify architectural patterns are still current best practice
- Check for known issues or breaking changes in referenced library versions

#### 2.4.4 Azure DevOps Ticket Alignment

- Do the Azure DevOps tickets match the implementation plan's task breakdown?
- Are there new tickets added since the plan was written?
- Are there tickets that were removed or changed scope?

### 2.5 Auto-Fix Discrepancies

For each discrepancy found in 2.4:

| Severity | Action |
|----------|--------|
| **Minor** (wrong line numbers, slight API changes, updated import paths) | Fix automatically in the plan file. No user notification needed. |
| **Medium** (deprecated API replaced with new equivalent, pattern evolved but same intent) | Fix automatically. Log the change for the Phase 2 summary. |
| **MAJOR** (architectural change, missing feature, broken core assumption, security concern) | **STOP and ask the user.** Present: what the plan assumed, what reality is, and your recommended fix. |

**After auto-fixing**, update the plan file on disk and commit:

```
docs(plans): revalidate <feature-name> implementation plan

Auto-fixed: [brief list of changes]
Refs: #[FEATURE_ID]
```

### 2.6 Revalidation Summary (informational only — no gate)

Log a brief summary to the console (do NOT wait for user confirmation unless MAJOR issues were found):

```
Plan Revalidation Complete:
- Design doc: [title] — [1-sentence summary]
- Implementation plan: [N tasks] covering [scope summary]
- Plan source: [pre-existing from /feature | generated in this session]
- Auto-fixes applied: [count] ([brief descriptions])
- Major issues: [none | BLOCKED — see above]
- Library docs verified: [list of libraries checked via Context7]
- Proceeding to Phase 3...
```

**If no MAJOR issues, proceed immediately to Phase 3. Do not wait for user input.**

### CHECKPOINT 2
```

- [ ] **Step 2: Commit**

```bash
git add commands/develop.md
git commit -m "feat(develop): add plan revalidation with Context7 re-verification (Phase 2)"
```

---

### Task 7: Fix develop.md — Quality Gates (Phases 6 and 7)

**Files:**
- Modify: `commands/develop.md`

- [ ] **Step 1: Insert Phase 6 after Phase 5**

After the existing Phase 5 (Build & Test) and its checkpoint, insert:

```markdown
## PHASE 6: CODE SIMPLIFICATION [MANDATORY QUALITY GATE]

**This phase is MANDATORY. You MUST NOT skip it. The feature is incomplete without code simplification.**

### 6.1 Summarize Context for Handoff

Prepare a summary of:
- Which files were created or modified in this feature
- The feature's purpose and architecture decisions
- Known areas of complexity

### 6.2 Run Code Simplifier

Invoke via the **`Agent` tool with `subagent_type: "pr-review-toolkit:code-simplifier"`**:

- Pass the list of modified files and feature context in the prompt
- The agent simplifies code for clarity, consistency, and maintainability
- It preserves all functionality while improving code quality

### 6.3 Apply Fixes

Apply suggestions from the code-simplifier. Focus on: removing unnecessary complexity, improving naming/readability, ensuring consistent patterns, removing dead code or unused imports.

### 6.4 Re-run Build & Tests

After simplification changes, verify nothing broke. Run the appropriate build + test commands for the project's `TECH_STACK`.

Fix any regressions.

### CHECKPOINT 6
```

- [ ] **Step 2: Insert Phase 7 after Phase 6**

```markdown
## PHASE 7: PR REVIEW [MANDATORY QUALITY GATE]

**This phase is MANDATORY. You MUST NOT skip it. The feature is incomplete without PR review.**

### 7.1 Run PR Review

Invoke via the **`Skill` tool with `skill: "pr-review-toolkit:review-pr"`**:

- Code quality and best practices
- Security vulnerabilities
- Logic errors and edge cases
- Adherence to project conventions

### 7.2 Apply Review Fixes

Address all HIGH and MEDIUM severity issues:

1. Fix each issue
2. Document why LOW severity issues were left (if any)

### 7.3 Run Code Simplifier Again (Second Pass)

After fixing PR review issues, invoke **`Agent` tool with `subagent_type: "pr-review-toolkit:code-simplifier"`** one more time to ensure fixes maintain code quality.

### 7.4 Final Build & Test Run

Run the complete build + test suite one last time. Use the appropriate commands for the project's `TECH_STACK`.

**All tests MUST pass before proceeding. Do not continue if any test fails.**

### CHECKPOINT 7
```

- [ ] **Step 3: Renumber existing Phase 8, 9, 10 if needed**

Verify the existing commit, close-tickets, and merge phases are numbered 8, 9, 10 respectively and their checkpoints match.

- [ ] **Step 4: Commit**

```bash
git add commands/develop.md
git commit -m "feat(develop): add mandatory quality gates — code simplification (Phase 6) and PR review (Phase 7)"
```

---

### Task 8: Fix develop.md — Phase 10 Merge, State Update, Enforcement Rules

**Files:**
- Modify: `commands/develop.md`

- [ ] **Step 1: Update Phase 10 with worktree cleanup and state update**

Rewrite Phase 10 to include:

```markdown
## PHASE 10: MERGE TO DEVELOP & WORKTREE CLEANUP

**Switch back to the main working directory for the merge.** The main directory is still on `develop`.

### 10.1 Return to main working directory and pull latest

```bash
cd "$(git worktree list | head -1 | awk '{print $1}')"
git pull origin develop
```

### 10.2 Merge the feature branch

```bash
git merge feature/[short-feature-name]
```

If there are merge conflicts: analyze, resolve, re-run build + tests. If conflicts are complex, ask the user.

### 10.3 Verify the merge

Run the build + test commands appropriate for the project's `TECH_STACK`.

**All tests MUST pass. If any test fails, fix before proceeding.**

### 10.4 Clean up temporary plan files

Delete the working plan files — Azure DevOps tickets are the permanent record:

```bash
rm -f docs/plans/<feature-name>-design.md docs/plans/<feature-name>.md
git add docs/plans/ && git commit -m "chore: remove temporary plan files for <feature-name>"
```

### 10.5 Remove worktree and delete the feature branch

```bash
FLAT_BRANCH=$(echo "feature/[short-feature-name]" | tr '/' '--')
git worktree remove ".worktrees/$FLAT_BRANCH" --force
git branch -d feature/[short-feature-name]
```

### 10.x Update Workflow State

If `.state.md` exists, update it:
- `step`: `develop`
- `status`: `ready-to-promote`
- `next_command`: `/staging`
- `last_command`: `/develop`
- `last_updated`: current ISO timestamp
- Append to History: `- [date time] /develop — status: completed (feature merged to develop)`

### CHECKPOINT 10 (FINAL)

Mark Phase 10 as `completed`. Call `TaskList` to confirm ALL 10 phases are completed.
```

- [ ] **Step 2: Add Workflow Enforcement Rules at the end of the file**

```markdown
## WORKFLOW ENFORCEMENT RULES

1. **Task list is the source of truth.** If pending phases remain, you are not done. Always call `TaskList` at checkpoints.
2. **Plan files are the implementation guide.** Follow them. If they don't exist, Phase 2 generates them. Once generated, follow them — don't reinvent the plan.
3. **Phases 6 and 7 are mandatory quality gates.** If you find yourself about to commit without having run them, STOP and go back.
4. **Never combine phases.** Each phase has its own checkpoint.
5. **Build + tests run 3 times minimum.** Phase 5 (initial), Phase 6 (after simplification), Phase 7 (after PR review fixes).
6. **Code Simplifier runs 2 times.** Phase 6 (first pass) and Phase 7 (after PR review fixes).
7. **Always work on a feature branch in a worktree.** Never commit directly to develop/master/main. Never switch the main working directory away from `develop`.
8. **Phase 10 merges to develop from the main directory.** Feature branch MUST be merged, worktree removed, and branch deleted.
9. **Plan deviations must be documented.** If you deviate from the plan, note what changed and why in the Azure DevOps ticket comments (Phase 9.1).
10. **Azure DevOps is the single source of truth.** All history, decisions, and deviations go into ticket comments — not local files.
11. **Plan files are temporary.** They exist only during development. Phase 10 cleans them up after merge.

## ERROR RECOVERY

If a phase fails critically:

1. **Do not skip the phase.** Stop and report the issue to the user.
2. **Ask the user** whether to: (a) continue debugging, (b) revert changes from that phase, or (c) proceed with a documented exception.
3. **Never proceed past a quality gate (Phase 6 or 7) with known failures.**
```

- [ ] **Step 3: Make all build/test references tech-stack-aware**

Search the entire file for hardcoded `npm run build`, `npm test`, `npm run test:e2e`, `npm install`, `npx tsc --noEmit`, and `npm run lint`. Replace each with a tech-stack-aware note:

```markdown
Run the appropriate build/test commands for the project's `TECH_STACK`:
- `nextjs`: `npm run build && npm run lint && npx tsc --noEmit`, then `npm test && npm run test:e2e`
- `dotnet`: `dotnet build`, then `dotnet test`
- `python`: `python -m pytest`

Or read the exact commands from the project's `CLAUDE.md`.
```

- [ ] **Step 4: Commit**

```bash
git add commands/develop.md
git commit -m "feat(develop): add worktree merge/cleanup, state update, enforcement rules, tech-stack-aware commands"
```

---

### Task 9: Update init-project.md and validate-env.md — Template Paths

**Files:**
- Modify: `commands/init-project.md`
- Modify: `commands/validate-env.md`

- [ ] **Step 1: Update init-project.md template references**

Find all references to `templates/` in `init-project.md` and replace with `${CLAUDE_PLUGIN_ROOT:-.}/templates/`. Specific locations:

In Step 4 (Scaffold Wiki):
```
For each wiki template file in `${CLAUDE_PLUGIN_ROOT:-.}/templates/wiki/`:
```

In Step 5 (Generate Azure Pipeline), the template path column:
```
| nextjs | hetzner | `${CLAUDE_PLUGIN_ROOT:-.}/templates/pipelines/azure-pipelines-nextjs-hetzner.yml` |
| nextjs | vercel | `${CLAUDE_PLUGIN_ROOT:-.}/templates/pipelines/azure-pipelines-nextjs-vercel.yml` |
| dotnet | azure | `${CLAUDE_PLUGIN_ROOT:-.}/templates/pipelines/azure-pipelines-dotnet-azure.yml` |
```

In Step 6 (Update CLAUDE.md):
```
Read the `${CLAUDE_PLUGIN_ROOT:-.}/templates/claude-md-wiki-block.md` template
```

In Step 8 (Initialize State File):
```
Copy `.state.md` template from `${CLAUDE_PLUGIN_ROOT:-.}/templates/.state.md.example`
```

- [ ] **Step 2: Add .env.claude gitignore step to init-project.md**

After Step 2 (Validate Azure DevOps Connection) and before Step 3, insert:

```markdown
## Step 2.5: Ensure .env.claude is Gitignored

`.env.claude` contains secrets (PAT token). Ensure it's not committed:

```bash
git check-ignore -q .env.claude 2>/dev/null || echo ".env.claude" >> .gitignore
```
```

- [ ] **Step 3: Update validate-env.md template reference**

Find the remediation message in the Load Configuration section:
```
echo "  → Create from template: cp templates/.env.claude.example .env.claude"
```

Replace with:
```
echo "  → Create from template: cp ${CLAUDE_PLUGIN_ROOT:-.}/templates/.env.claude.example .env.claude"
```

- [ ] **Step 4: Add dependency checks to validate-env.md**

After the existing Check 8 (State File Check), add a new section:

```markdown
### 9. Plugin Dependency Checks

| Check | Command | Pass | Fail |
|-------|---------|------|------|
| Azure CLI installed | `az --version 2>/dev/null` | [PASS] | [FAIL] → Install Azure CLI: https://aka.ms/installazurecli |
| DevOps extension | `az extension show --name azure-devops 2>/dev/null` | [PASS] | [FAIL] → Run: `az extension add --name azure-devops` |
| `superpowers` plugin | Check if superpowers skills are loadable | [PASS] | [WARN] → Install: `/plugin install superpowers@claude-plugins-official` |
| `pr-review-toolkit` plugin | Check if pr-review-toolkit skills are loadable | [PASS] | [WARN] → Install: `/plugin install pr-review-toolkit@claude-plugins-official` |
```

Update the output format to include the new section.

- [ ] **Step 5: Commit**

```bash
git add commands/init-project.md commands/validate-env.md
git commit -m "fix: update template paths to use CLAUDE_PLUGIN_ROOT, add dependency checks"
```

---

### Task 10: Add LICENSE File

**Files:**
- Create: `LICENSE`

- [ ] **Step 1: Create MIT license file**

```
MIT License

Copyright (c) 2026 AndrianopoulosGeo

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 2: Commit**

```bash
git add LICENSE
git commit -m "chore: add MIT license"
```

---

### Task 11: Rewrite README.md for Marketplace Distribution

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Rewrite README.md**

Replace the entire contents with marketplace-focused documentation:

```markdown
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

1. Create `.env.claude` at your repo root:
   ```bash
   cp ${CLAUDE_PLUGIN_ROOT}/templates/.env.claude.example .env.claude
   ```
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
| `/azure-devops-lifecycle:wiki` | Manage project documentation wiki |

### Orchestration
| Command | Description |
|---------|-------------|
| `/azure-devops-lifecycle:next` | Auto-advance to next step in current workflow |

## Configuration

All commands read from `.env.claude` at your project root. See `templates/.env.claude.example` for the full schema.

### Required Fields
- `AZURE_DEVOPS_ORG` — Your Azure DevOps organization
- `AZURE_DEVOPS_PROJECT` — Project name
- `AZURE_DEVOPS_PAT` — Personal Access Token
- `DEPLOY_TARGET` — `hetzner` | `azure` | `vercel`
- `TECH_STACK` — `nextjs` | `dotnet` | `python`

### Optional Fields
- `STAGING_URL`, `PRODUCTION_URL` — Environment URLs
- `BRANCH_STRATEGY` — Defaults to `gitflow`
- `STAGING_PIPELINE_ID`, `PRODUCTION_PIPELINE_ID` — Auto-populated by init-project

## License

MIT
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: rewrite README for marketplace distribution"
```

---

### Task 12: Set Up Marketplace Repo

**Files:**
- Create (in separate repo): `claude-marketplace/.claude-plugin/marketplace.json`
- Create (in separate repo): `claude-marketplace/README.md`

- [ ] **Step 1: Clone the marketplace repo**

```bash
cd /c/Users/andri/Dev
git clone https://github.com/AndrianopoulosGeo/claude-marketplace.git
cd claude-marketplace
```

- [ ] **Step 2: Create marketplace manifest**

```bash
mkdir -p .claude-plugin
```

Write `.claude-plugin/marketplace.json`:

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
      "description": "Full development lifecycle automation with Azure DevOps — feature planning, TDD implementation, staging, release, hotfix, and wiki management with Gitflow branching",
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

- [ ] **Step 3: Create marketplace README**

```markdown
# Andrianopoulos Marketplace

Claude Code plugin marketplace.

## Plugins

| Plugin | Description |
|--------|-------------|
| [azure-devops-lifecycle](https://github.com/AndrianopoulosGeo/azure-devops-lifecycle) | Full development lifecycle automation with Azure DevOps |

## Usage

```bash
# Add this marketplace
/plugin marketplace add AndrianopoulosGeo/claude-marketplace

# Browse available plugins
/plugin

# Install a plugin
/plugin install azure-devops-lifecycle@andrianopoulos-marketplace
```
```

- [ ] **Step 4: Commit and push**

```bash
git add .claude-plugin/marketplace.json README.md
git commit -m "feat: initialize marketplace with azure-devops-lifecycle plugin"
git push origin main
```

---

### Task 13: Push Plugin Repo to GitHub

**Files:**
- No file changes — git operations only

- [ ] **Step 1: Create GitHub repo** (if not already created)

```bash
cd "/c/Users/andri/Dev/Claude Skills"
gh repo create AndrianopoulosGeo/azure-devops-lifecycle --public --source=. --remote=github
```

Or if the repo already exists, just add the remote:

```bash
git remote add github https://github.com/AndrianopoulosGeo/azure-devops-lifecycle.git
```

- [ ] **Step 2: Push all commits**

```bash
git push github master
```

---

### Task 14: Validate Installation

- [ ] **Step 1: Add the marketplace**

```bash
/plugin marketplace add AndrianopoulosGeo/claude-marketplace
```

- [ ] **Step 2: Install the plugin**

```bash
/plugin install azure-devops-lifecycle@andrianopoulos-marketplace
```

- [ ] **Step 3: Verify commands are available**

Try running `/azure-devops-lifecycle:validate-env` in a test project with `.env.claude` configured. Verify:
- Command loads and executes
- Template paths resolve correctly via `${CLAUDE_PLUGIN_ROOT}`
- All checks run (even if some fail due to missing setup)

- [ ] **Step 4: Verify all commands are listed**

```bash
/plugin
```

Confirm `azure-devops-lifecycle` appears with all 10 commands listed.
