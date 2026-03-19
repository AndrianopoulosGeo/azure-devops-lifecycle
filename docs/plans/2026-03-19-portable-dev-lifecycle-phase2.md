# Phase 2: Lifecycle Commands — `/staging`, `/release`, `/quick-fix`, `/hotfix`

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the 4 lifecycle commands that complete the three development tracks (full feature, quick fix, hotfix).

**Architecture:** Each command is a markdown prompt file in `commands/`. Each reads `.env.claude` for config and `.state.md` for workflow state. Each updates `.state.md` at completion. Commands orchestrate existing superpowers skills.

**Tech Stack:** Claude Code commands (markdown prompts), Azure CLI, git

**Working Directory:** `C:\Users\andri\Dev\Claude Skills`

---

## File Structure

| File | Action | Responsibility |
|------|--------|---------------|
| `commands/staging.md` | Create | Promote develop → staging, verify deployment |
| `commands/release.md` | Create | Promote staging → master, tag, verify production |
| `commands/quick-fix.md` | Create | Fast track: ticket → branch → fix → merge to develop |
| `commands/hotfix.md` | Create | Emergency: branch from master → fix → merge to both |

---

## Task 1: Create `/staging` Command

**Files:**
- Create: `commands/staging.md`

- [ ] **Step 1: Write the command**
- [ ] **Step 2: Verify line count (~120-150 lines)**
- [ ] **Step 3: Commit**

---

## Task 2: Create `/release` Command

**Files:**
- Create: `commands/release.md`

- [ ] **Step 1: Write the command**
- [ ] **Step 2: Verify line count (~130-160 lines)**
- [ ] **Step 3: Commit**

---

## Task 3: Create `/quick-fix` Command

**Files:**
- Create: `commands/quick-fix.md`

- [ ] **Step 1: Write the command**
- [ ] **Step 2: Verify line count (~140-170 lines)**
- [ ] **Step 3: Commit**

---

## Task 4: Create `/hotfix` Command

**Files:**
- Create: `commands/hotfix.md`

- [ ] **Step 1: Write the command**
- [ ] **Step 2: Verify line count (~140-170 lines)**
- [ ] **Step 3: Commit**

---

## Task 5: Update `/develop` and `/feature` for State Management

**Working Directory:** `C:\Users\andri\Dev\MDT dynamics`

Update the existing `/develop` and `/feature` commands to:
1. Read from `.env.claude` (develop.md still uses old `.env` pattern)
2. Write `.state.md` at completion

- [ ] **Step 1: Update develop.md config + state write**
- [ ] **Step 2: Update feature.md config + state write**
- [ ] **Step 3: Commit**

---

## Task 6: Push Claude Skills and Verify

- [ ] **Step 1: Push Claude Skills repo**
- [ ] **Step 2: Verify all commands exist**

---

## Summary

| Task | Deliverable | Repo |
|------|-------------|------|
| 1 | `/staging` command | Claude Skills |
| 2 | `/release` command | Claude Skills |
| 3 | `/quick-fix` command | Claude Skills |
| 4 | `/hotfix` command | Claude Skills |
| 5 | Update `/develop` + `/feature` for state management | MDT Dynamics |
| 6 | Push and verify | Claude Skills |

**Next:** Phase 3 — `/wiki` command + Phase 4 knowledge management
