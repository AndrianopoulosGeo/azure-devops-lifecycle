---
allowed-tools: [Read, Bash, Glob, Grep]
---

# /publish-wiki — Publish Wiki to Azure DevOps

> **Expert Voice:** DevOps Engineer — syncs local documentation to the remote wiki, handles page creation and updates, ensures consistency.

You publish the local project wiki (`docs/wiki/`) to Azure DevOps as a **code wiki**. Each repo in the Azure DevOps project gets its own wiki, scoped to the repo's `docs/wiki/` folder. This avoids namespace collisions in multi-repo projects.

**Usage:**
- `/publish-wiki` — Publish all wiki sections to Azure DevOps
- `/publish-wiki <section>` — Publish a specific section (e.g., `architecture`)

The `$ARGUMENTS` parameter contains the optional section filter.

## Step 1: Load Configuration

```bash
export AZURE_DEVOPS_PAT=$(grep AZURE_DEVOPS_PAT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_ORG=$(grep AZURE_DEVOPS_ORG .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_PROJECT=$(grep AZURE_DEVOPS_PROJECT .env.claude | cut -d '=' -f2)
export AZURE_DEVOPS_EXT_PAT=$AZURE_DEVOPS_PAT
az devops configure --defaults organization=https://dev.azure.com/$AZURE_DEVOPS_ORG project="$AZURE_DEVOPS_PROJECT"
```

## Step 2: Detect Repository Identity

Determine the current repo's name and ID in Azure DevOps:

```bash
# Get the Azure DevOps remote URL
git remote -v | grep dev.azure.com | head -1
```

Extract the repo name from the remote URL:
- `https://org@dev.azure.com/org/project/_git/RepoName` → `RepoName`
- `https://dev.azure.com/org/project/_git/RepoName` → `RepoName`

Then get the repo ID:

```bash
az repos show --repository "$REPO_NAME" --output json --query "id" -o tsv
```

Store both `REPO_NAME` and `REPO_ID` for subsequent steps.

## Step 3: Ensure Code Wiki Exists

Check if a code wiki already exists for this repo:

```bash
az devops wiki list --output json
```

Parse the response and look for a wiki where:
- `type` is `codeWiki`
- `repositoryId` matches `$REPO_ID`

**If a matching code wiki exists:**
- Store its `name` and `id`

**If NO code wiki exists for this repo, create one:**

```bash
az devops wiki create \
  --name "$REPO_NAME.wiki" \
  --type codeWiki \
  --repository "$REPO_ID" \
  --mapped-path "/docs/wiki" \
  --branch "master" \
  --output json
```

This maps the wiki directly to the `docs/wiki/` folder in the repo. Azure DevOps will serve the markdown files from that folder as wiki pages.

Store the wiki name for subsequent commands.

**Note on branch:** Use `master` as the default. If the repo uses `main` as the default branch, detect it:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "master"
```

## Step 4: Push Wiki Content

Since code wikis are served directly from the repo's branch, the wiki content is already in `docs/wiki/`. The key steps are:

### 4.1 Ensure docs/wiki/ is committed and pushed

```bash
# Check for uncommitted wiki changes
git status docs/wiki/
```

If there are uncommitted changes in `docs/wiki/`:

```bash
git add docs/wiki/
git commit -m "docs(wiki): update wiki content"
```

### 4.2 Push to Azure DevOps remote

```bash
git push origin $(git branch --show-current)
```

The code wiki automatically reflects whatever is on the mapped branch. No page-by-page API calls needed — pushing the files is the publish action.

### 4.3 Strip frontmatter for clean rendering

Code wikis render markdown files directly. YAML frontmatter (`---` blocks) will show as visible text. Create a clean copy for the wiki:

For each file in `docs/wiki/`:
1. Check if it has YAML frontmatter
2. If yes, strip the frontmatter block (everything between the first two `---` lines)
3. Stage the change

```bash
for file in docs/wiki/*.md; do
  if head -1 "$file" | grep -q "^---$"; then
    sed -i '1{/^---$/!q;};1,/^---$/d' "$file"
  fi
done
git add docs/wiki/
git diff --cached --quiet || git commit -m "docs(wiki): strip frontmatter for Azure DevOps rendering"
git push origin $(git branch --show-current)
```

**Important:** This permanently removes frontmatter from the wiki files. If frontmatter is needed for other purposes (e.g., `/wiki status` uses it), consider keeping frontmatter and accepting that it renders visibly, or maintaining a separate publishing branch.

## Step 5: Verify Wiki is Accessible

```bash
az devops wiki page show --path "/" --wiki "$WIKI_NAME" --output json 2>&1
```

If this returns content, the wiki is live. If it fails, check:
- The mapped path exists on the mapped branch
- The branch has been pushed to the Azure DevOps remote
- There's at least one `.md` file in `docs/wiki/`

## Step 6: Set Page Order

Create or update a `.order` file in `docs/wiki/` to control page ordering in the Azure DevOps wiki sidebar:

```bash
cat > docs/wiki/.order << 'EOF'
index
architecture
api-reference
data-model
services
testing
deployment
configuration
conventions
EOF
```

Commit and push if the `.order` file was created or changed:

```bash
git add docs/wiki/.order
git diff --cached --quiet || git commit -m "docs(wiki): set page order"
git push origin $(git branch --show-current)
```

## Step 7: Summary Report

```
/publish-wiki completed:
==========================================
Wiki type: Code Wiki (per-repo)
Wiki name: $WIKI_NAME
Repo:      $REPO_NAME
Branch:    $BRANCH
Mapped:    /docs/wiki

URL: https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_wiki/wikis/$WIKI_NAME

Published pages:
  [PUBLISHED]  index.md            → Home
  [PUBLISHED]  architecture.md     → Architecture
  [PUBLISHED]  api-reference.md    → API Reference
  [PUBLISHED]  data-model.md       → Data Model
  [PUBLISHED]  services.md         → Services
  [PUBLISHED]  testing.md          → Testing
  [PUBLISHED]  deployment.md       → Deployment
  [PUBLISHED]  configuration.md    → Configuration
  [PUBLISHED]  conventions.md      → Conventions
  [SKIPPED]    <file>.md           → (status: template)

Result: N published | M skipped
==========================================

Other wikis in this project:
  - OtherRepo.wiki (code wiki, repo: OtherRepo)
  - ...
```

List other wikis in the project so the user sees the multi-repo wiki landscape.

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| Repo not found | Remote URL doesn't match any Azure DevOps repo | Check git remotes point to this Azure DevOps project |
| Wiki already exists | A code wiki with same name exists for different repo | Use `$REPO_NAME.wiki` naming to avoid collisions |
| 401 Unauthorized | PAT expired or missing permissions | Check PAT has Code Read/Write and Wiki Read/Write |
| Branch not found | Mapped branch doesn't exist on remote | Push the branch first |
| No files in mapped path | `docs/wiki/` is empty or not committed | Run `/wiki auto` to generate content, then commit |

If any step fails, report the error and suggest remediation.
