---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent]
description: Commit changes with no AI attribution and auto-update context docs
---

# /commit — Smart Commit

> **Expert Voice:** Disciplined Developer — commits clean, human-authored changes and keeps project documentation in sync automatically.

You create commits that look entirely human-written and automatically update context management docs when the code changes warrant it. This is the **single commit entry point** — all other commands delegate to this.

**Usage:**
- `/commit` — Commit all staged/unstaged changes
- `/commit <message>` — Commit with a specific message hint

The `$ARGUMENTS` parameter contains an optional commit message hint.

---

## Step 1: Analyze Changes

Read the current state:

```bash
git status
git diff HEAD
git branch --show-current
git log --oneline -10
```

If there are no changes to commit (no staged, unstaged, or untracked files), stop:
> "Nothing to commit — working tree is clean."

---

## Step 2: Classify Changed Files

From the diff, classify what was changed into categories:

| Category | Pattern | Triggers Doc Check |
|----------|---------|-------------------|
| **Source code** | `src/`, `app/`, `lib/`, `pages/`, `components/`, `*.cs`, `*.py`, `*.ts`, `*.tsx`, `*.js`, `*.jsx` | `CLAUDE.md`, `docs/architecture.md` |
| **Tests** | `*test*`, `*spec*`, `__tests__/`, `e2e/`, `tests/` | `docs/TESTING.md` |
| **API routes** | `*/api/*`, `*Controller*`, `*route*` | `docs/wiki/api-reference.md`, `docs/architecture.md` |
| **Config/infra** | `*.yml`, `*.yaml`, `Dockerfile`, `*.csproj`, `package.json`, `tsconfig*` | `CLAUDE.md` |
| **Project structure** | New directories, moved files | `CLAUDE.md` (Project Structure section) |
| **Docs only** | `docs/`, `*.md` | No additional checks needed |

---

## Step 3: Auto-Update Context Docs

Based on the classification from Step 2, check and update the relevant docs. **Only update sections that are actually stale** — do not rewrite unchanged sections.

### 3.1 CLAUDE.md

Check if any of these sections need updating:

- **Project Structure** — if new directories were added or files were moved
- **Key Files Reference** — if new key files were introduced (models, API routes, components)
- **Build & Test Commands** — if `package.json` scripts, `*.csproj`, or build config changed

**How to check**: Read the current `CLAUDE.md` section, compare against the actual project state (file tree, package.json scripts). If they diverge, update the section.

If `CLAUDE.md` doesn't exist, skip — don't create one here (that's `/update-claude-md`'s job).

### 3.2 docs/architecture.md

Check if changes affect:

- **Component hierarchy** — new components, renamed components, moved components
- **Data flow** — new API routes, changed data patterns
- **Directory conventions** — new directories, restructured folders

**How to check**: Read the current `docs/architecture.md`, compare the documented structure against the changed files. If a new component/route/pattern was introduced that isn't documented, add it.

If the file doesn't exist, skip.

### 3.3 docs/TESTING.md

Check if changes affect:

- **Test patterns** — new test utilities, mocking patterns, test setup changes
- **Test structure** — new test directories, renamed test files
- **Test commands** — changes to test runner config

**How to check**: Read `docs/TESTING.md`, compare against actual test files and configuration. Update if new patterns were introduced.

If the file doesn't exist, skip.

### 3.4 docs/wiki/ pages

Check if changes affect documented features:

- **API changes** → update `docs/wiki/api-reference.md` if it exists
- **New components/pages** → update `docs/wiki/architecture.md` if it exists
- **Configuration changes** → update `docs/wiki/configuration.md` if it exists
- **Deployment changes** → update `docs/wiki/deployment.md` if it exists

**How to check**: For each relevant wiki page, read it and verify the documented information still matches reality. Only update pages that exist and have stale content.

### 3.5 Stage Doc Updates

If any docs were updated:

```bash
git add CLAUDE.md docs/architecture.md docs/TESTING.md docs/wiki/ 2>/dev/null
```

Only stage files that were actually modified.

---

## Step 4: Create Commit

### 4.1 Determine Commit Message

If `$ARGUMENTS` contains a message hint, use it as the base. Otherwise, analyze the changes and generate a message following **Conventional Commits** format:

```
<type>(<scope>): <summary>

<body — explain WHY, not WHAT>

Refs: #<ticket-id>
```

**Types**: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `style`, `perf`, `ci`, `build`

If the changes include both code and auto-updated docs, the commit type should reflect the **code changes** (not the docs). The docs update is incidental.

### 4.2 Ticket Reference

Look for a ticket ID from:
1. The current branch name (e.g., `feature/123-foo` → `#123`, `fix/456-bar` → `#456`)
2. `$ARGUMENTS` if it contains a `#number`
3. `.state.md` if it has a `ticket_id` field

If found, append `Refs: #<ticket-id>` to the commit body.

### 4.3 AI Attribution Rules

**HARD RULES — NEVER violate these:**

- NEVER include `Co-Authored-By` with any AI reference
- NEVER include `Generated by`, `Created by`, or `Assisted by` with AI/Claude/Anthropic
- NEVER mention Claude, Claude Code, AI, Anthropic, LLM, or GPT in commit messages
- Commit messages must read as **entirely human-written**

### 4.4 Stage and Commit

```bash
git add -A
git commit -m "<message>"
```

---

## Step 5: Post-Commit Verification

After the commit is created, verify the commit message is clean:

```bash
git log -1 --format="%B"
```

Scan the output for forbidden patterns: `Claude`, `claude`, `Anthropic`, `anthropic`, `AI`, `GPT`, `LLM`, `Co-Authored-By`, `Generated by`, `Assisted by`, `noreply@anthropic`.

If ANY forbidden pattern is found, amend the commit to remove it:

```bash
git commit --amend -m "<cleaned message>"
```

---

## Step 6: Summary

```
/commit — Done
===============
Branch: <branch>
Commit: <short hash> <message first line>
Docs updated: <list of updated docs, or "none">
```
