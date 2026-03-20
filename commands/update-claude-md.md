---
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent]
---

# /update-claude-md — Update Project Context in CLAUDE.md

> **Expert Voice:** Documentation Architect — ensures Claude always has the right context to make informed decisions. Writes precise, actionable reference documentation.

You analyze the codebase and update CLAUDE.md with comprehensive project context. Your goal: after running this command, Claude should know exactly where to look for any kind of information in this project.

**Usage:**
- `/update-claude-md` — Full analysis and update
- `/update-claude-md section` — Update a specific section (e.g., `models`, `architecture`, `conventions`)

The `$ARGUMENTS` parameter contains the optional section filter.

## Step 1: Load Configuration

```bash
export TECH_STACK=$(grep TECH_STACK .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
```

## Step 2: Read Current CLAUDE.md

Read the existing `CLAUDE.md`. If it doesn't exist, create one with a basic header:
```markdown
# {{PROJECT_NAME}}
```

Identify which sections already exist and what needs updating.

## Step 3: Analyze the Codebase

Run these analyses to discover project structure. Use Agent tool to parallelize independent scans:

### 3.1 Project Structure Discovery

```bash
# Get directory tree (top 3 levels)
find . -maxdepth 3 -type d ! -path './.git/*' ! -path './node_modules/*' ! -path './.worktrees/*' | sort

# Count files by type
find . -type f ! -path './.git/*' ! -path './node_modules/*' | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20
```

### 3.2 Models & Data Layer

Discover where data models live and what ORM/database is used:

```bash
# Prisma
find . -name "schema.prisma" 2>/dev/null

# EF Core / .NET
find . -name "*.cs" -exec grep -l "DbContext\|DbSet\|\[Table\]\|\[Key\]" {} \; 2>/dev/null | head -20

# SQLAlchemy / Django
find . -name "*.py" -exec grep -l "Base\|models.Model\|Column\|relationship" {} \; 2>/dev/null | head -20

# TypeORM / Drizzle
grep -r "Entity\|entity\|schema\|drizzle\|typeorm" --include="*.ts" -l 2>/dev/null | head -20
```

### 3.3 API Layer

```bash
# Next.js API routes
find . -path "*/api/*" -name "route.ts" -o -name "route.js" 2>/dev/null

# Express/Fastify routes
grep -r "app.get\|app.post\|router\.\|@Get\|@Post\|@Controller" --include="*.ts" --include="*.js" -l 2>/dev/null | head -20

# .NET controllers
find . -name "*Controller.cs" 2>/dev/null
```

### 3.4 Component Architecture (for frontend)

```bash
# React/Next.js components
find . -path "*/components/*" -name "*.tsx" -o -name "*.jsx" 2>/dev/null | head -30

# Pages/routes
find . -path "*/app/*" -name "page.tsx" -o -name "page.jsx" 2>/dev/null
find . -path "*/pages/*" -name "*.tsx" 2>/dev/null | head -20
```

### 3.5 Configuration & Environment

```bash
# Config files
ls -la .env* tsconfig.json next.config.* vitest.config.* jest.config.* playwright.config.* appsettings.json *.csproj pyproject.toml 2>/dev/null

# Environment variable usage
grep -r "process.env\.\|Environment.GetEnvironmentVariable\|os.environ\|os.getenv" --include="*.ts" --include="*.tsx" --include="*.cs" --include="*.py" -h 2>/dev/null | sort -u | head -30
```

### 3.6 Testing Setup

```bash
# Test files
find . -name "*.test.*" -o -name "*.spec.*" 2>/dev/null | head -20

# Test config
cat package.json 2>/dev/null | grep -A5 '"test"'
```

### 3.7 Existing Documentation

```bash
# Check what docs already exist
find . -name "*.md" -path "*/docs/*" 2>/dev/null
find . -name "CLAUDE.md" -o -name "README.md" -o -name "CONTRIBUTING.md" 2>/dev/null
```

## Step 4: Generate Context Sections

Based on the analysis, update CLAUDE.md with these sections. **Preserve any existing content** — only add or update sections, never remove user-written content.

### Section: Project Structure

```markdown
## Project Structure

| Directory | Purpose |
|-----------|---------|
| `src/app/` | Next.js app router pages and API routes |
| `src/components/` | React components (sections/, ui/, layout/) |
| `src/lib/` | Shared utilities, constants, types |
| ... | (discovered from analysis) |
```

### Section: Key Files Reference

```markdown
## Key Files Reference

| Need to understand... | Look at these files |
|----------------------|-------------------|
| Data models & schema | `prisma/schema.prisma`, `src/models/` |
| API endpoints | `src/app/api/`, see `docs/wiki/api-reference.md` |
| UI components | `src/components/`, see `docs/wiki/architecture.md` |
| Test patterns | `src/__tests__/`, see `docs/wiki/testing.md` |
| Build & deploy | `azure-pipelines.yml`, `Dockerfile`, see `docs/wiki/deployment.md` |
| Coding standards | `docs/wiki/conventions.md`, `.eslintrc`, `tsconfig.json` |
| Architecture decisions | `docs/decisions/` (ADRs) |
| Environment config | `.env.claude`, see `docs/wiki/configuration.md` |
```

### Section: Build & Test Commands

```markdown
## Build & Test Commands

| Action | Command |
|--------|---------|
| Install | `npm install` |
| Dev server | `npm run dev` |
| Build | `npm run build` |
| Lint | `npm run lint` |
| Type check | `npx tsc --noEmit` |
| Unit tests | `npm test` |
| E2E tests | `npm run test:e2e` |
```

(Adapt commands based on TECH_STACK — read from package.json scripts, .csproj, or pyproject.toml)

### Section: Conventions

```markdown
## Conventions

- **Commits**: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`)
- **Branches**: Gitflow (`feature/`, `fix/`, `hotfix/`)
- **Naming**: (discovered from codebase — camelCase, PascalCase, etc.)
- **Testing**: (discovered — TDD, test file co-location, etc.)
```

## Step 5: Present Changes and Confirm

Show the user what sections were added or updated. Ask:
> "Here's what I've added to CLAUDE.md. Look correct? (y/n)"

If yes, save. If no, let them specify changes.

## Step 6: Commit

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with project context and reference guide"
```
