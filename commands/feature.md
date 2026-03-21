---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent, Skill]
---

Create a fully planned feature: brainstorm the design, write implementation plans, then create Azure DevOps work items. Produces 2 plan files that `/develop` consumes.

**Design mode**: When the feature involves UI/frontend work, pass `--design` or include "design" / "UI" / "frontend" / "page" / "component" in your description to activate the **Design Toolchain** — this triggers frontend-design, ui-ux-pro-max, and 21st magic MCP tools during brainstorming and planning.

---

## STEP 1: AUTHENTICATE

Read Azure DevOps configuration from CLAUDE.md (look for Organization, Project, and PAT location). Then authenticate:

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

---

## STEP 2: GATHER INPUT

If `$ARGUMENTS` contains a description, use it. Otherwise ask the user to describe what they want to build.

Then ask (use `AskUserQuestion` with options):

1. **Starting level**: What is the top-level work item to create?
   - Epic (full breakdown: Epic > Features > Stories > Tasks)
   - Feature (breakdown: Feature > Stories > Tasks)
   - User Story (breakdown: Story > Tasks)
   - Attach to existing parent (provide parent work item ID)

2. **Parent ID** (if attaching): Which existing work item should this be linked under?

3. **Priority**: 1 (Critical), 2 (High), 3 (Medium), 4 (Low) — default 2

4. **Feature type**: Is this a UI/design feature?
   - Yes — activates Design Toolchain (frontend-design, ui-ux-pro-max, 21st magic, component inspiration)
   - No — standard backend/infra/cross-cutting feature

**Auto-detect**: If `$ARGUMENTS` contains `--design`, or keywords like "design", "UI", "frontend", "page", "component", "layout", "dashboard", "form", "widget", "screen", skip this question and activate Design Toolchain automatically.

---

## STEP 3: LOAD ARCHITECTURAL REFERENCES

Read the project's **architectural and convention reference docs**. These define HOW we build — the brainstorming design MUST conform to these rules.

### 3.1 Architecture & Conventions (REQUIRED — read all)

| File | What it provides |
|------|-----------------|
| `CLAUDE.md` | Project conventions, code standards, naming rules, test commands |
| `docs/architecture.md` | **PRIMARY reference** — component hierarchy, server vs client components, data flow, styling architecture, API architecture, directory conventions |
| `docs/decisions/` | Architecture Decision Records (ADRs) — past decisions and their rationale (e.g., Next.js App Router, Tailwind v4, Framer Motion) |
| `docs/TESTING.md` | **Testing reference** — test runners, commands, file structure, mocking patterns, E2E conventions, CI pipeline |

> **Portability note:** These paths refer to the **target project**, not the plugin. Read whatever exists — not all projects have all files. Let `TECH_STACK` from `.env.claude` guide which config files to look for (e.g., `package.json` for nextjs, `.csproj` for dotnet, `pyproject.toml` for python).

### 3.2 Project Configuration (read for context — read whatever exists)

| File | What it provides |
|------|-----------------|
| `package.json` or `*.csproj` or `pyproject.toml` | Dependencies, scripts, tech stack |
| Framework config (`next.config.ts`, `appsettings.json`, etc.) | Framework-specific configuration |
| `tsconfig.json` | TypeScript configuration (if applicable) |
| Test config (`vitest.config.ts`, `playwright.config.ts`, etc.) | Test runner configuration (if applicable) |

### 3.3 What to extract for brainstorming

After reading these files, prepare a **concise architectural brief** covering:

- **Component rules**: Server vs. client component guidelines (from architecture.md)
- **Patterns to follow**: Styling conventions (Tailwind utilities, `cn()` helper, CSS variables), animation patterns (Framer Motion, `prefers-reduced-motion`)
- **Testing strategy**: What test types are required (unit, integration, E2E) and how they're structured (from TESTING.md)
- **Existing components**: What already exists in `src/components/` that the new feature might reuse or extend
- **Directory conventions**: Where files go (`src/app/`, `src/components/sections/`, `src/components/ui/`, `src/lib/`)
- **Data flow**: Static data in `src/lib/constants.ts`, environment variables pattern

---

## STEP 4: BRAINSTORM (invoke superpowers:brainstorming)

**Invoke the `superpowers:brainstorming` skill** to run the full collaborative design flow. This skill handles:
- Exploring project context (files, docs, recent commits)
- Asking clarifying questions one at a time
- Proposing 2-3 approaches with trade-offs
- Presenting the design section by section with user approval gates
- Writing the spec to `docs/superpowers/specs/`
- Running spec review loop
- User review gate

**Pass this context to brainstorming:**
- The user's feature description from Step 2
- The **architectural brief** from Step 3.3 — so the brainstorming skill knows:
  - Which component patterns to follow (server vs. client, Tailwind, Framer Motion)
  - What tests are required and how to structure them (from TESTING.md)
  - What existing components to reuse or extend
  - Which directory conventions to follow
- Whether Design Toolchain is active (from Step 2)

The brainstorming skill should use this architectural context to:
1. **Constrain proposals** — only propose approaches that fit the existing architecture
2. **Map components to structure** — every proposed component must have a clear home (section component? UI primitive? layout component? hook?)
3. **Include test strategy** — the design must specify what test types each component needs (unit, integration, E2E)
4. **Reference existing patterns** — "this follows the same pattern as [existing component X]"

**Wait for brainstorming to complete and produce an approved spec before proceeding.**

---

## STEP 4.5: CONTEXT GATHERING (MANDATORY — always runs)

**This step runs for EVERY feature, not just UI features.** Before writing any implementation plan, gather current documentation, best practices, and tool context for all technologies the feature touches.

### 4.5.1 Fetch Library Documentation (Context7 — mandatory)

Use `resolve-library-id` + `query-docs` to fetch up-to-date documentation for **every library and framework** the feature touches. This is NOT optional — plans written without verified library docs risk using deprecated or non-existent APIs.

Examples by tech stack:
- **nextjs**: Next.js, React, Tailwind CSS, Framer Motion, any ORM (Prisma, Drizzle)
- **dotnet**: ASP.NET Core, Entity Framework, any NuGet packages the feature introduces
- **python**: FastAPI/Django/Flask, SQLAlchemy, any pip packages the feature introduces
- Any new libraries the feature introduces regardless of stack

### 4.5.2 Best Practices Research (WebSearch — targeted)

Search for current best practices on specific patterns the feature requires:
- New integration patterns not yet used in the project
- Complex architectural patterns (e.g., real-time features, file upload, caching, auth strategies)
- Security considerations for the feature's domain
- Accessibility (WCAG) guidelines if UI elements are involved

### 4.5.3 MCP Coding Tools (use ALL available)

Discover and use **any MCP server tools available in the session** that can provide coding context. These tools fill the context window with project-specific intelligence before writing the plan.

**How to discover:** Check which MCP tools are available (they appear in system reminders). Use any that are relevant to the feature's tech stack.

Common examples:
| MCP Tool | When to use |
|----------|-------------|
| `context7` | Always — library documentation |
| `pyright-lsp` | Python features — type checking, symbol resolution, diagnostics |
| `typescript-lsp` | TypeScript features — type info, diagnostics, completions |
| `microsoft-docs` | .NET / Azure features — official Microsoft documentation |
| `playwright` | Features that need browser testing context |
| Any language server | Read type signatures, find usages, check diagnostics for files the feature will modify |

**The goal:** By the end of this step, you should have verified, current documentation for every API you'll use in the implementation plan. No guessing.

### 4.5.4 Update the Spec

Incorporate research findings into the approved brainstorming spec:
- Add a **Libraries Verified** section listing libraries + versions confirmed
- Add relevant API signatures, patterns, or constraints discovered
- Flag any deprecated patterns or breaking changes that affect the design

---

## STEP 4.6: DESIGN TOOLCHAIN (only if UI/design feature)

**Skip this step if the feature is NOT a UI/design feature.**

When Design Toolchain is active, enrich the spec with **visual design research** using available design MCP tools:

1. **`ui-ux-pro-max`** — Invoke the `Skill` tool with `skill: "ui-ux-pro-max"` for:
   - Design system decisions (color palette, typography, spacing)
   - Component design patterns (buttons, modals, cards, forms, charts)
   - Style direction (glassmorphism, minimalism, dark mode, etc.)

2. **`frontend-design`** — Invoke the `Skill` tool with `skill: "frontend-design:frontend-design"` for:
   - Production-grade component code generation
   - Distinctive, polished UI that avoids generic AI aesthetics
   - Creative design approaches

3. **`21st magic`** — Use `mcp__magic__21st_magic_component_inspiration` for:
   - Component inspiration and reference designs
   - Modern UI patterns and trends
   - Use `mcp__magic__21st_magic_component_builder` for building components with modern patterns

Update the spec with:
- **Visual Design** section with chosen styles, colors, typography
- **Component Specifications** with exact component hierarchy and props
- **Responsive Breakpoints** and layout behavior

**Do NOT proceed until the user approves the enriched design spec.**

---

## STEP 5: CREATE DESIGN DOCUMENT

Derive `<feature-name>` from the feature title (kebab-case, e.g., `blog-section`, `service-detail-pages`).

Save the approved design as a **temporary working file**:

```
docs/plans/<feature-name>-design.md
```

Include:
- Feature title and one-sentence goal
- Architecture decisions and approach chosen (from brainstorming)
- Component breakdown
- Data flow and integration points
- Edge cases and error handling strategy
- Testing strategy overview (referencing docs/TESTING.md patterns)
- Constraints and open questions resolved during brainstorm

**These files are working documents** — they'll be consumed by `/develop` and then cleaned up after merge. Azure DevOps tickets are the permanent record.

---

## STEP 6: CREATE IMPLEMENTATION PLAN (superpowers:writing-plans pattern)

**Context gathering was completed in Step 4.5.** All library docs, best practices, and MCP tool context are already available. Use them to write the plan with verified, current API usage.

### 6.1 Write the Implementation Plan

Using the approved design, write a detailed implementation plan. **Assume the implementing engineer has zero codebase context** — document everything they need.

Save as a **temporary working file**:

```
docs/plans/<feature-name>.md
```

### Plan Header (REQUIRED)

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence]

**Architecture:** [2-3 sentences — component structure, data flow, animation approach]

**Tech Stack:** Next.js, React, Framer Motion, Tailwind CSS, TypeScript

**Design Doc:** `docs/plans/<feature-name>-design.md`

**Testing Reference:** `docs/TESTING.md`

**Library Versions Verified:** [List libraries + versions confirmed via Context7]

---
```

### Task Structure

Each task follows TDD with bite-sized steps (2-5 minutes each):

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

**Build/test commands vary by TECH_STACK:**
- `nextjs`: `npm run build`, `npm test`, `npm run test:e2e`
- `dotnet`: `dotnet build`, `dotnet test`
- `python`: `python -m pytest`, `python -m pytest e2e/`

Alternatively, read the exact build/test commands from the project's `CLAUDE.md` if they are documented there.

### Plan Requirements

- **Exact file paths** matching Next.js project structure (`src/app/`, `src/components/sections/`, `src/components/ui/`, `src/components/layout/`, `src/lib/`, `src/hooks/`)
- **Complete code** in the plan (not "add validation" — show the actual code)
- **Dependency order**: shared utils/types → components → pages → API routes → animations/polish
- **DRY, YAGNI, TDD** — frequent commits
- **Testing follows docs/TESTING.md**: Unit tests in `src/__tests__/`, E2E tests in `e2e/`, mocking patterns from `src/__tests__/setup.tsx`
- **Verified APIs**: All library API usage must match the docs fetched in Step 6.0 — no guessing deprecated or non-existent methods

---

## STEP 7: FETCH EXISTING BOARD STATE & IDENTIFY BLOCKERS

Query the current board to understand what already exists, avoid duplicates, and **detect potential blocking dependencies**:

```bash
# Get all active work items
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [System.AssignedTo], [System.Tags] FROM workitems WHERE [System.TeamProject] = '$AZURE_DEVOPS_PROJECT' AND [System.State] <> 'Closed' AND [System.State] <> 'Removed' ORDER BY [System.WorkItemType], [System.Id]" -o table
```

### 7.1 Check for Overlapping Work Items

Check for overlapping or related work items. Warn the user if similar items already exist.

### 7.2 Identify Blocking Dependencies

Analyze the existing active work items against the new feature's implementation plan to find **potential blockers** — tickets that must complete first or that work on shared code/components in parallel:

**Look for conflicts in these areas:**
- **Shared files**: Other tickets modifying the same files (components, pages, utils, styles) the new feature touches
- **Shared data**: Other tickets changing constants, types, API routes, or data structures the new feature depends on
- **Shared infrastructure**: Other tickets modifying build config, CI/CD, shared hooks, or middleware
- **UI conflicts**: Other tickets working on the same page sections, layout areas, or navigation elements
- **Feature dependencies**: Other tickets implementing APIs, components, or utilities that the new feature needs

For each potential blocker found, determine the relationship:
- **Blocked by** (must finish first): e.g., the new feature needs an API that another ticket is building
- **Blocks** (other ticket needs this first): e.g., the new feature creates a shared component another ticket is waiting for
- **Related / parallel risk** (no hard dependency but merge conflict risk): e.g., both modify the same file

### 7.3 Present Dependencies

If potential blockers are found, present them to the user before proceeding:

```
⚠ Potential Dependencies Detected:
├── [BLOCKED BY] #123 "Create user API endpoint" (Active, assigned to @dev)
│   └── Reason: New feature needs the /api/users endpoint this ticket creates
├── [PARALLEL RISK] #456 "Redesign navigation bar" (Active, assigned to @dev2)
│   └── Reason: Both modify src/components/layout/Navbar.tsx — merge conflicts likely
└── [RELATED] #789 "Add analytics tracking" (New, unassigned)
    └── Reason: Both add event handlers to the same page section
```

**Ask the user** which dependencies to formally link as blocking relations on the board.

---

## STEP 8: DESIGN THE WORK ITEM HIERARCHY

Use the brainstorm design (Step 4) and implementation plan (Step 6) to create informed, accurate work items.

### Epic (if starting from Epic)
- Title: Strategic goal or initiative name
- Description: Business value, success metrics, scope boundaries

### Feature
- Title: Deliverable functionality name
- Description: What capability this delivers, which components/pages it touches, key technical decisions
- **Embed the design summary and implementation plan overview in the description** — the Feature ticket is the permanent record of what was planned and why

### User Story
- Title: "As a [user role], I want [goal] so that [benefit]"
- Description: Behavior, Acceptance Criteria (checklist), Scope, Edge Cases

### Task
- Title: Concrete implementation step (imperative verb)
- Description: What, Where (file paths), Pattern, Tests
- **Align tasks with the implementation plan's task breakdown from Step 6**

### Rules

- Stories are user-facing, Tasks are developer-facing
- Every Story has acceptance criteria (testable checklist)
- Tasks reference concrete files/modules from the implementation plan
- Build order: shared utils → components → pages → API routes → animations → tests
- Testing is not optional — every Story includes testing tasks
- No duplicate scope between Stories

---

## STEP 9: PRESENT THE HIERARCHY

Show the full tree structure:

```
[Epic] Title (if applicable)
├── [Feature] Title
│   ├── [Story] As a user, I want...
│   │   ├── [Task] Create component for...
│   │   ├── [Task] Add animation for...
│   │   ├── [Task] Write unit tests for...
│   │   └── [Task] Write E2E tests for...
│   └── [Story] ...
└── ...
```

**Wait for user confirmation before creating.** The user may want to modify items.

---

## STEP 10: CREATE WORK ITEMS & LINK DEPENDENCIES

Create items top-down, linking each child to its parent:

```bash
# Create top-level item
az boards work-item create --type "<type>" --title "<title>" --description "<description>" --fields "Microsoft.VSTS.Common.Priority=<priority>"

# Create child and link to parent
az boards work-item create --type "<child-type>" --title "<title>" --description "<description>" --fields "Microsoft.VSTS.Common.Priority=<priority>"
az boards work-item relation add --id <CHILD_ID> --relation-type "Parent" --target-id <PARENT_ID>
```

**Order**: Epic → Features → User Stories → Tasks (parents before children)

### 10.1 Add Blocking Relations

After creating all work items, add the dependency links the user approved in Step 7.3:

```bash
# Link "blocked by" — this new item is blocked by an existing item
az boards work-item relation add --id <NEW_ITEM_ID> --relation-type "Predecessor" --target-id <BLOCKING_ITEM_ID>

# Link "blocks" — this new item blocks an existing item
az boards work-item relation add --id <NEW_ITEM_ID> --relation-type "Successor" --target-id <BLOCKED_ITEM_ID>

# Link "related" — parallel work with merge conflict risk (informational)
az boards work-item relation add --id <NEW_ITEM_ID> --relation-type "Related" --target-id <RELATED_ITEM_ID>
```

**Note**: Link dependencies at the appropriate level — typically Feature-to-Feature or Story-to-Story, not at the Task level unless the dependency is very specific.

---

## STEP 11: SUMMARY

Display:

1. **Full tree view** with all created work item IDs
2. **Stats**: Total items created per type
3. **Board link**: `https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_boards/board`
4. **Working plan files** (temporary — will be cleaned up after `/develop` completes):
   ```
   Design:         docs/plans/<feature-name>-design.md
   Implementation: docs/plans/<feature-name>.md
   ```
5. **Next step**: "Run `/develop <feature-id>` to start implementing this feature. The develop command will read the plan files and clean them up after merge."

---

## STEP 12: Update Workflow State

If `.state.md` exists (or create it from template), update:
- `track`: `feature`
- `step`: `feature`
- `branch`: current branch or `develop`
- `ticket_id`: the Feature work item ID created above
- `started_at`: current ISO timestamp
- `last_command`: `/feature`
- `status`: `ready-to-promote`
- `next_command`: `/develop`
- Append to History: `- [date time] /feature — status: completed (Feature #ID planned, tickets created)`
