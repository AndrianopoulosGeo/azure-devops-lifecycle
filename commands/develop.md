---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill, TaskCreate, TaskUpdate, TaskList]
---

# Automated Feature Development Workflow

Execute the full development lifecycle for a feature. This command reads plan files (design doc + implementation plan) if they exist, or **generates them automatically** from the Azure DevOps feature description and ticket hierarchy.

**This is a two-level orchestration**: an outer workflow (tracked via TaskCreate/TaskUpdate) drives 10 phases. Plan files guide implementation — they can come from `/feature` or be generated inline in Phase 2.

**If `$ARGUMENTS` contains a feature ID, use that feature. Otherwise, auto-detect the next feature to develop.**

---

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

---

## PHASE 1: SETUP & CONTEXT LOADING

**Mark Phase 1 as `in_progress` via `TaskUpdate`.**

### 1.1 Authenticate to Azure DevOps

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

### 1.2 Fetch the target feature

If `$ARGUMENTS` contains a feature ID, use it. Otherwise, query the next "New" feature by StackRank:

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [Microsoft.VSTS.Common.Priority], [Microsoft.VSTS.Common.StackRank] FROM workitems WHERE [System.TeamProject] = '$AZURE_DEVOPS_PROJECT' AND [System.WorkItemType] = 'Feature' AND [System.State] = 'New' ORDER BY [Microsoft.VSTS.Common.StackRank] ASC" --output table
```

**Display to the user**: "Next feature to develop: #[ID] - [Title]". Ask for confirmation before proceeding.

### 1.3 Fetch all child work items (Stories + Tasks)

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [System.Description] FROM workitems WHERE [System.TeamProject] = '$AZURE_DEVOPS_PROJECT' AND [System.Parent] = [FEATURE_ID] ORDER BY [System.Id] ASC" --output table
```

Then for each Story, fetch its child Tasks. Build a full tree of everything that needs to be implemented.

### 1.4 Locate and read the 2 plan files

Search for temporary working plan files created by `/feature`. Derive `<feature-name>` from the feature title (kebab-case) and search:

```bash
ls docs/plans/<feature-name>*.md 2>/dev/null
```

If not found by name, list all plan files and ask the user to confirm:

```bash
ls -la docs/plans/*.md
```

**Read whatever plan files exist:**
- **Design doc**: `docs/plans/<feature-name>-design.md` — architecture decisions, approaches, component breakdown, edge cases
- **Implementation plan**: `docs/plans/<feature-name>.md` — task-by-task TDD implementation steps with exact file paths and code

**Note what is found vs missing.** Do NOT stop here. Phase 2 will handle missing plans:
- Both files found → Phase 2 revalidates them
- Design doc found, implementation plan missing → Phase 2 generates the implementation plan
- Neither found → Phase 2 generates both from the Azure DevOps feature description + ticket hierarchy

### 1.5 Load Architectural References

Read these files — they MUST inform all implementation decisions:

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Project conventions, code standards, test commands |
| `docs/architecture.md` | **PRIMARY reference** — component hierarchy, server vs client components, data flow, styling, API architecture, directory conventions |
| `docs/decisions/` | Architecture Decision Records — past decisions and constraints |
| `docs/TESTING.md` | **Testing reference** — test runners, commands, file structure, mocking patterns, E2E conventions, CI pipeline |

### CHECKPOINT 1

---

## PHASE 2: PLAN REVALIDATION & GENERATION (AUTONOMOUS)

**This phase ensures both plan files exist, are valid, and are verified against current documentation.** It runs autonomously — no user intervention required unless MAJOR conflicts are found.

It handles three scenarios:

### Scenario A: Both plan files exist (happy path)

Skip to **2.4 Revalidate** below.

### Scenario B: Design doc exists, implementation plan missing

Generate the implementation plan from the design doc + Azure DevOps tickets.

### Scenario C: Neither plan file exists

Generate both from the Azure DevOps feature description + child work item hierarchy.

---

### 2.1 Generate Design Doc (only if missing — Scenario C)

If no design doc exists, create one at `docs/plans/<feature-name>-design.md` by synthesizing:

- The **Feature work item description** from Azure DevOps (fetched in Phase 1.3)
- The **Story descriptions** and their acceptance criteria
- The **project context** loaded in Phase 1.5 (architecture.md, decisions/)

The design doc should cover:
- Problem statement and goal (from the Feature description)
- Architecture approach (informed by architecture.md patterns)
- Component breakdown (derived from the Stories)
- Data flow and integration points
- Edge cases and error handling strategy
- Testing strategy overview (referencing docs/TESTING.md)

**Keep it concise** — this is a synthesis of existing approved tickets, not a brainstorming session.

### 2.2 Generate Implementation Plan (if missing — Scenarios B and C)

Create the implementation plan at `docs/plans/<feature-name>.md` using the `superpowers:writing-plans` pattern.

**Input sources (in priority order):**

1. **Design doc** (from 2.1 or existing) — architecture decisions, component breakdown
2. **Azure DevOps Task descriptions** — concrete implementation steps with file paths
3. **Project context** — architecture.md patterns, existing component conventions
4. **Testing reference** — docs/TESTING.md for test structure and mocking patterns
5. **Context7 library docs** — fetch up-to-date docs for libraries referenced in the design (Next.js, Framer Motion, Tailwind CSS, etc.)

**Plan structure** — follow the same format as `/feature` Step 6:

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence]

**Architecture:** [2-3 sentences — component structure, data flow, animation approach]

**Tech Stack:** Next.js, React, Framer Motion, Tailwind CSS, TypeScript

**Design Doc:** `docs/plans/<feature-name>-design.md`

**Testing Reference:** `docs/TESTING.md`

---
```

Each task follows TDD with bite-sized steps:

```markdown
### Task N: [Component Name]

**Files:**
- Create: `src/components/[component].tsx`
- Create: `src/app/[route]/page.tsx`
- Test (unit): `src/__tests__/[component].test.tsx`
- Test (E2E): `e2e/[feature].spec.ts`

**Step 1: Write the failing test**
[Complete test code — follow patterns from docs/TESTING.md]

**Step 2: Run test to verify it fails**
Run: npm test -- [test-file]
Expected: FAIL with "[reason]"

**Step 3: Write minimal implementation**
[Complete implementation code]

**Step 4: Run test to verify it passes**
Run: npm test -- [test-file]
Expected: PASS

**Step 5: Commit**
[exact git commands]
```

**Plan requirements:**
- **Read the actual source files** referenced by the Azure DevOps Tasks before writing plan steps — use real line numbers, real function signatures, real import paths
- **Exact file paths** matching architecture.md directory conventions
- **Complete code** in the plan (not "add validation" — show the actual code)
- **Dependency order**: shared utils/types → components → pages → API routes → animations/polish
- **Test code follows docs/TESTING.md** — Vitest for unit tests in `src/__tests__/`, Playwright for E2E in `e2e/`, mocking patterns from `src/__tests__/setup.tsx`
- **Align tasks with Azure DevOps Task work items** — each plan task should map to one or more Azure DevOps Tasks
- **DRY, YAGNI, TDD** — frequent commits

### 2.3 Commit Generated Plan Files

If any plan files were generated, commit them:

```
docs(plans): add <feature-name> implementation plan

Refs: #[FEATURE_ID]
```

### 2.4 Autonomous Revalidation (NO user intervention)

**This step runs automatically and fixes issues on its own.** The goal is to verify that EVERYTHING the plan claims is still accurate against the current codebase and current library documentation.

#### 2.4.1 Codebase Verification

Check each claim in the plan against reality:

- **File paths**: Do all referenced files exist? Are the line ranges still accurate?
- **Function signatures**: Do the functions/methods referenced in the plan still have the same signatures?
- **Import paths**: Are all import paths valid in the current codebase?
- **Existing patterns**: Do the patterns the plan follows match what the codebase currently uses? (e.g., if other components use a new pattern, the plan should too)
- **Recent changes**: Have any relevant files been modified since the plan was written? (`git log --since` on referenced files)

#### 2.4.2 Documentation & Tool Re-Verification

Re-verify all library and framework usage in the plan using **every available MCP coding tool**:

**Context7 (mandatory):** Use `resolve-library-id` + `query-docs` for every library the plan references. Cross-check that API usage matches CURRENT docs. Flag deprecated methods, changed signatures, or new recommended approaches.

**MCP coding tools (use ALL available):** Check which MCP tools are available in the session and use any relevant ones:
- Language servers (pyright-lsp, typescript-lsp, etc.) — verify type signatures, check diagnostics on referenced files
- `microsoft-docs` — for .NET/Azure API verification
- Any other coding MCP tools — use them to validate the plan's assumptions

**WebSearch (targeted):** For complex or newer patterns, verify:
- Architectural patterns are still current best practice
- No known issues or breaking changes in referenced library versions
- Accessibility patterns match current WCAG recommendations (if applicable)

#### 2.4.4 Azure DevOps Ticket Alignment

- Do the Azure DevOps tickets match the implementation plan's task breakdown?
- Are there new tickets added since the plan was written?
- Are there tickets that were removed or changed scope?

### 2.5 Auto-Fix Discrepancies

**This is the key difference from the old flow: fix issues automatically instead of asking the user.**

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

### 2.6 Quality Evaluation (SOLID, Best Practices, Architecture)

After revalidation and auto-fixes, evaluate the **entire implementation plan** for engineering quality. Use MCP coding tools, WebSearch, and Context7 to assess:

#### 2.6.1 SOLID Principles Check

Review every component, class, and module in the plan against SOLID:

| Principle | What to check |
|-----------|--------------|
| **Single Responsibility** | Does each component/module do one thing? Are concerns separated (data fetching vs. rendering vs. business logic)? |
| **Open/Closed** | Are components extensible without modification? Are hooks, props, or config used for variation instead of conditionals? |
| **Liskov Substitution** | Can interfaces/abstractions be swapped without breaking consumers? |
| **Interface Segregation** | Are interfaces/props lean? No component forced to depend on things it doesn't use? |
| **Dependency Inversion** | Do high-level modules depend on abstractions, not concrete implementations? |

#### 2.6.2 Best Practices Verification

Use **WebSearch** and **MCP coding tools** to verify the plan follows current best practices:

- **Framework-specific patterns**: Are we using the recommended patterns for the framework version? (e.g., Server Components vs Client Components in Next.js, async patterns in .NET, type hints in Python)
- **Security**: Input validation, auth patterns, SQL injection prevention, XSS prevention — are they handled correctly?
- **Performance**: Are there obvious performance anti-patterns? (N+1 queries, unnecessary re-renders, missing caching, unoptimized data fetching)
- **Error handling**: Is error handling consistent and appropriate? Are edge cases covered?
- **Testability**: Is the code structured for easy testing? Are dependencies injectable?

#### 2.6.3 Architecture Alignment

Cross-check the plan against `docs/architecture.md` and `docs/decisions/`:
- Does the plan follow the project's established patterns?
- If it introduces new patterns, are they justified and documented?
- Are directory conventions respected?

#### 2.6.4 Auto-Fix Quality Issues

For each issue found:

| Severity | Action |
|----------|--------|
| **Minor** (naming, small pattern improvements) | Fix automatically in the plan. |
| **Medium** (SOLID violation with clear fix, missing error handling) | Fix automatically. Log for summary. |
| **MAJOR** (fundamental architecture issue, security concern) | **STOP and ask the user.** |

Commit any fixes:

```
docs(plans): quality evaluation fixes for <feature-name>

Applied: [brief list of SOLID/best-practice improvements]
Refs: #[FEATURE_ID]
```

### 2.7 Revalidation Summary (informational only — no gate)

Log a brief summary to the console (do NOT wait for user confirmation unless MAJOR issues were found):

```
Plan Revalidation Complete:
- Design doc: [title] — [1-sentence summary]
- Implementation plan: [N tasks] covering [scope summary]
- Plan source: [pre-existing from /feature | generated in this session]
- Auto-fixes applied: [count] ([brief descriptions])
- Quality evaluation: [N issues found, M auto-fixed]
- SOLID compliance: [pass | N violations fixed]
- Major issues: [none | BLOCKED — see above]
- Library docs verified: [list of libraries checked]
- MCP tools used: [list of tools used for verification]
- Proceeding to Phase 3...
```

**If no MAJOR issues, proceed immediately to Phase 3. Do not wait for user input.**

### CHECKPOINT 2

---

## PHASE 3: ENVIRONMENT PREPARATION (WORKTREE-BASED)

**All implementation work happens in an isolated git worktree.** Your main working directory stays on `develop` so VS Code is never disrupted.

### 3.1 Ensure worktree directory is git-ignored

```bash
git check-ignore -q .worktrees 2>/dev/null || echo ".worktrees/" >> .gitignore && git add .gitignore && git commit -m "chore: add .worktrees to gitignore"
```

### 3.2 Create feature branch in a worktree

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

Run the appropriate install command for the project's `TECH_STACK`:
- `nextjs`: `npm install`
- `dotnet`: `dotnet restore`
- `python`: `pip install -r requirements.txt` (or equivalent)

### 3.4 Verify clean baseline

Run the appropriate build/test commands for the project's `TECH_STACK`:
- `nextjs`: `npm run build && npm run lint && npx tsc --noEmit`, then `npm test && npm run test:e2e`
- `dotnet`: `dotnet build`, then `dotnet test`
- `python`: `python -m pytest`

Or read the exact commands from the project's `CLAUDE.md`.

**If tests fail:** Report failures and ask the user whether to proceed or investigate. Do NOT continue with a broken baseline.

### 3.5 Set Feature to "Active"

```bash
az boards work-item update --id [FEATURE_ID] --state "Active" --output none
```

### 3.6 Fill context with MCP coding tools

Use **every available MCP coding tool** to load context before implementation:

**Context7 (mandatory):** Use `resolve-library-id` + `query-docs` to fetch up-to-date documentation for every library/framework the implementation plan references.

**MCP coding tools (use ALL available):** Check which MCP tools are available in the session and use any relevant ones:
- Language servers (pyright-lsp, typescript-lsp, etc.) — load type info, check diagnostics for files the plan will modify
- `microsoft-docs` — for .NET/Azure features
- Any other coding MCP tools — use them to build implementation context

**WebSearch (targeted):** If the plan introduces patterns, libraries, or integrations not previously used in the project, search for current best practices.

### CHECKPOINT 3

**IMPORTANT:** From this point forward (Phases 4-9), ALL file edits, builds, tests, and commits happen inside the worktree at `$WORKTREE_PATH`. The main working directory remains untouched on `develop`.

---

## PHASE 4: IMPLEMENTATION

**REMINDER: All Phase 4 work happens inside the worktree directory (`$WORKTREE_PATH`), NOT the main working directory.**

### 4.1 Set Stories to "Active"

```bash
az boards work-item update --id [STORY_ID] --state "Active" --output none
```

### 4.2 Implement Feature (following the implementation plan)

**Follow the implementation plan from Phase 1.4 strictly.** The plan contains task-by-task TDD steps with exact file paths, code, and test commands.

For each task in the plan:

1. Set the corresponding Azure DevOps Task to "Active":
   ```bash
   az boards work-item update --id [TASK_ID] --state "Active" --output none
   ```

2. **Follow the plan's steps exactly**: write failing test → verify it fails → implement → verify it passes → commit

3. **Per-task verification with MCP tools.** Before writing implementation code for each task:
   - Use **Context7** to verify API signatures you're about to use
   - Use **language server MCP tools** (pyright-lsp, typescript-lsp, etc.) if available — check types, get diagnostics, resolve symbols in files you're modifying
   - Use **WebSearch** if the task involves a pattern, integration, or architecture decision not fully covered in the plan
   - If tools reveal a better approach, adopt it and note the deviation

4. **Verify quality**: Before closing each task, confirm:
   - Components follow SOLID principles (single responsibility, dependency inversion)
   - Code follows the project's established patterns from architecture.md
   - Test patterns match docs/TESTING.md conventions
   - No security anti-patterns (unvalidated input, injection risks)

5. Close the task:
   ```bash
   az boards work-item update --id [TASK_ID] --state "Closed" --output none
   ```

### 4.3 Handle Plan Deviations

If during implementation you discover the plan needs adjustment (wrong assumptions, missing steps, unexpected complexity):

1. Note the deviation clearly
2. If minor (wrong line numbers, slight API differences): adapt and continue
3. If major (architectural change, missing feature, broken assumption): stop and ask the user before proceeding

### 4.4 Update Acceptance Criteria

After implementation, update each User Story with acceptance criteria:

```bash
az boards work-item update --id [STORY_ID] --description "<existing description>\n\n## Acceptance Criteria\n- [ ] Criterion 1\n- [ ] Criterion 2\n..."
```

### CHECKPOINT 4

**STOP AND VERIFY**: Call `TaskList`. You should see Phases 5, 6, 7, 8, 9, and 10 still pending. There are **6 more phases** to complete. Implementation is NOT the end of the workflow.

---

## PHASE 5: BUILD & TEST

### 5.1 Build, typecheck, and lint

Run the appropriate build/test commands for the project's `TECH_STACK` as separate calls to isolate failures:
- `nextjs`: `npm run build && npm run lint && npx tsc --noEmit`, then `npm test && npm run test:e2e`
- `dotnet`: `dotnet build`, then `dotnet test`
- `python`: `python -m pytest`

Or read the exact commands from the project's `CLAUDE.md`.

Fix any compilation errors, type errors, or lint violations before proceeding to tests.

### 5.2 Run All Tests

Run the appropriate build/test commands for the project's `TECH_STACK`:
- `nextjs`: `npm run build && npm run lint && npx tsc --noEmit`, then `npm test && npm run test:e2e`
- `dotnet`: `dotnet build`, then `dotnet test`
- `python`: `python -m pytest`

Or read the exact commands from the project's `CLAUDE.md`.

### 5.3 Fix Any Failures

If tests fail: analyze, fix, re-run failing tests, repeat until all pass.

### 5.4 Report Test Results

Display a summary: total tests, passed, failed, coverage (if available).

### CHECKPOINT 5

**CRITICAL — DO NOT STOP HERE.** You MUST see Phases 6, 7, 8, 9, and 10 still pending. The feature is NOT done. Proceed immediately to Phase 6.

---

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

After simplification changes, verify nothing broke. Run the appropriate build/test commands for the project's `TECH_STACK`:
- `nextjs`: `npm run build && npm run lint && npx tsc --noEmit`, then `npm test && npm run test:e2e`
- `dotnet`: `dotnet build`, then `dotnet test`
- `python`: `python -m pytest`

Or read the exact commands from the project's `CLAUDE.md`.

Fix any regressions.

### CHECKPOINT 6

---

## PHASE 7: PR REVIEW [MANDATORY QUALITY GATE]

**This phase is MANDATORY. You MUST NOT skip it. The feature is incomplete without PR review.**

### 7.1 Run PR Review

Invoke via the **`Skill` tool with `skill: "pr-review-toolkit:review-pr"`**:

- Code quality and best practices
- Security vulnerabilities
- Logic errors and edge cases
- Adherence to project conventions (Next.js, Tailwind, Framer Motion patterns)

### 7.2 Apply Review Fixes

Address all HIGH and MEDIUM severity issues:

1. Fix each issue
2. Document why LOW severity issues were left (if any)

### 7.3 Run Code Simplifier Again (Second Pass)

After fixing PR review issues, invoke **`Agent` tool with `subagent_type: "pr-review-toolkit:code-simplifier"`** one more time to ensure fixes maintain code quality.

### 7.4 Final Build & Test Run

Run the complete build + test suite one last time. Run the appropriate build/test commands for the project's `TECH_STACK`:
- `nextjs`: `npm run build && npm run lint && npx tsc --noEmit`, then `npm test && npm run test:e2e`
- `dotnet`: `dotnet build`, then `dotnet test`
- `python`: `python -m pytest`

Or read the exact commands from the project's `CLAUDE.md`.

**All tests MUST pass before proceeding. Do not continue if any test fails.**

### CHECKPOINT 7

---

## PHASE 8: COMMIT

### 8.1 Create Commits

Invoke via the **`Skill` tool with `skill: "commit"`**.

Group related changes into logical commits by invoking `/commit` multiple times with targeted staging:

1. Shared types, utils, and lib
2. Components and hooks
3. Pages and layouts
4. API routes
5. Animations and polish
6. Tests

The `/commit` skill handles: Conventional Commits format, AI-attribution stripping, ticket references from the branch name, and auto-updating context management docs (`CLAUDE.md`, `docs/architecture.md`, `docs/TESTING.md`, `docs/wiki/`).

### CHECKPOINT 8

---

## PHASE 9: CLOSE TICKETS & UPDATE KNOWLEDGE

### 9.1 Add Completion Comments to Work Items

Before resolving each work item, add a brief comment summarizing what was accomplished.

**For each Task** — add a comment:

```bash
az boards work-item update --id [TASK_ID] --discussion "Done. [1-2 sentence summary]" --output none
```

**For each User Story** — add a comment:

```bash
az boards work-item update --id [STORY_ID] --discussion "Completed. [1-2 sentences: what was delivered, any deviation from the plan]" --output none
```

**For the Feature** — add a comment:

```bash
az boards work-item update --id [FEATURE_ID] --discussion "Resolved. [2-3 sentences: what the feature delivers, key implementation decisions]" --output none
```

**Comment rules:**
- Keep comments **very brief** — 1-3 sentences max
- If the implementation **deviated** from the plan files, mention what changed and why
- **Never mention Claude, AI, or automated tools** in comments

### 9.2 Close User Stories

```bash
az boards work-item update --id [STORY_ID] --state "Resolved" --output none
```

### 9.3 Resolve the Feature

```bash
az boards work-item update --id [FEATURE_ID] --state "Resolved" --output none
```

### 9.4 Update the Main Epic

Check if all Features under the parent Epic are resolved. If so, update the Epic's state.

### 9.5 Update Knowledge

**Azure DevOps is the single source of truth for project history.** The ticket comments from 9.1 serve that purpose.

Only update these **living architectural docs** if the feature changed the architecture:

1. **Update `docs/architecture.md`** — only if new components, layers, or patterns were introduced
2. **Update `docs/decisions/`** — only if significant architectural decisions were made (new ADRs)
3. **Update `docs/TESTING.md`** — only if new test patterns, mocks, or conventions were introduced

### 9.6 Commit Knowledge Updates (only if docs were changed)

```
docs: update architecture for <feature-name>

Refs: #[FEATURE_ID]
```

### CHECKPOINT 9

---

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

Run the appropriate build/test commands for the project's `TECH_STACK`:
- `nextjs`: `npm run build && npm run lint && npx tsc --noEmit`, then `npm test && npm run test:e2e`
- `dotnet`: `dotnet build`, then `dotnet test`
- `python`: `python -m pytest`

Or read the exact commands from the project's `CLAUDE.md`.

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

---

## COMPLETION

Display a summary:

- **Feature**: #[ID] - [Title] -> Resolved
- **Plan files**: Used and cleaned up (Azure DevOps tickets are the permanent record)
- **Branch**: `feature/[name]` -> merged to `develop`, worktree removed, branch deleted
- **Stories completed**: [count] / [total]
- **Tasks completed**: [count] / [total]
- **Plan deviations**: [none / documented in Azure DevOps ticket comments]
- **Tests**: [passed] / [total] (coverage %)
- **Code quality passes**: Code Simplifier (2x) + PR Review (1x)
- **Commits created**: [count]
- **Architectural docs updated**: [architecture.md / decisions/ / TESTING.md / none]
- **Next feature**: #[ID] - [Title] (or "All features complete!")

---

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
12. **Testing follows docs/TESTING.md.** All test code must follow the patterns, structure, and conventions documented there.

## ERROR RECOVERY

If a phase fails critically:

1. **Do not skip the phase.** Stop and report the issue to the user.
2. **Ask the user** whether to: (a) continue debugging, (b) revert changes from that phase, or (c) proceed with a documented exception.
3. **Never proceed past a quality gate (Phase 6 or 7) with known failures.**
