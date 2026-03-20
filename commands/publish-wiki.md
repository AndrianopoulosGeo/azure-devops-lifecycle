---
allowed-tools: [Read, Bash, Glob, Grep]
---

# /publish-wiki — Publish Wiki to Azure DevOps

> **Expert Voice:** DevOps Engineer — syncs local documentation to the remote wiki, handles page creation and updates, ensures consistency.

You publish the local project wiki (`docs/wiki/`) to Azure DevOps Wiki. Each markdown file becomes a wiki page, preserving the structure and all Mermaid diagrams.

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

## Step 2: Ensure Wiki Exists

Check if a project wiki already exists:

```bash
az devops wiki list --output json
```

Parse the response:
- If a wiki exists, store its `name` and `id` for later use
- If NO wiki exists, create one:

```bash
az devops wiki create --name "$AZURE_DEVOPS_PROJECT" --type projectWiki --output json
```

Store the wiki name for subsequent commands.

## Step 3: Determine Pages to Publish

If `$ARGUMENTS` specifies a section:
- Publish only `docs/wiki/<section>.md`
- If the file doesn't exist, stop with an error

If no arguments (publish all):
- List all `.md` files in `docs/wiki/`
- Skip files with `status: template` in frontmatter (nothing to publish)
- Skip `index.md` — it will be handled as the root page

## Step 4: Prepare Content for Azure DevOps

For each wiki file to publish:

### 4.1 Read the local file

```bash
cat docs/wiki/<section>.md
```

### 4.2 Strip YAML frontmatter

Remove the `---` delimited frontmatter block from the content. Azure DevOps Wiki doesn't use YAML frontmatter — it would render as visible text.

### 4.3 Define the wiki page path

Map local filenames to Azure DevOps wiki paths:

| Local File | Wiki Path | Page Title |
|-----------|-----------|------------|
| `docs/wiki/index.md` | `/` | Home |
| `docs/wiki/architecture.md` | `/Architecture` | Architecture |
| `docs/wiki/api-reference.md` | `/API-Reference` | API Reference |
| `docs/wiki/data-model.md` | `/Data-Model` | Data Model |
| `docs/wiki/services.md` | `/Services` | Services & Integrations |
| `docs/wiki/testing.md` | `/Testing` | Testing |
| `docs/wiki/deployment.md` | `/Deployment` | Deployment |
| `docs/wiki/configuration.md` | `/Configuration` | Configuration |
| `docs/wiki/conventions.md` | `/Conventions` | Conventions |

## Step 5: Publish Pages

For each page, try to update first (page exists), then create if it doesn't:

### 5.1 Write content to a temp file (avoids shell escaping issues)

```bash
# Strip frontmatter and write to temp file
sed '1{/^---$/!q;};1,/^---$/d' docs/wiki/<section>.md > /tmp/wiki-page-content.md
```

### 5.2 Try to update the page

```bash
az devops wiki page update \
  --path "<wiki-path>" \
  --wiki "$WIKI_NAME" \
  --file-path "/tmp/wiki-page-content.md" \
  --comment "Update <section> from local wiki" \
  --version <etag> \
  --output json 2>&1
```

To get the current version (etag) for update:

```bash
az devops wiki page show --path "<wiki-path>" --wiki "$WIKI_NAME" --output json 2>&1
```

Extract the `eTag` field from the response. If the page doesn't exist, the show command will return an error.

### 5.3 If page doesn't exist, create it

```bash
az devops wiki page create \
  --path "<wiki-path>" \
  --wiki "$WIKI_NAME" \
  --file-path "/tmp/wiki-page-content.md" \
  --comment "Add <section> from local wiki" \
  --output json
```

### 5.4 Handle parent pages

If publishing to a nested path (e.g., `/Architecture/Diagrams`), ensure parent pages exist first. Create empty parent pages if needed.

## Step 6: Publish Index as Home Page

If publishing all pages, also publish `docs/wiki/index.md` as the wiki root page (`/`):

```bash
sed '1{/^---$/!q;};1,/^---$/d' docs/wiki/index.md > /tmp/wiki-page-content.md
az devops wiki page update --path "/" --wiki "$WIKI_NAME" --file-path "/tmp/wiki-page-content.md" --comment "Update wiki home page" --version <etag> --output json 2>&1
```

## Step 7: Set Page Order

After all pages are published, set the page ordering to match the logical reading flow:

1. Home (index)
2. Architecture
3. API Reference
4. Data Model
5. Services
6. Testing
7. Deployment
8. Configuration
9. Conventions

```bash
# Azure DevOps Wiki uses page order via the .order file in the wiki git repo
# This is handled automatically by the wiki system based on creation order
# If manual ordering is needed, it can be set via the wiki git repo directly
```

## Step 8: Summary Report

```
/publish-wiki completed:
==========================================
Wiki: $WIKI_NAME
URL:  https://dev.azure.com/$AZURE_DEVOPS_ORG/$AZURE_DEVOPS_PROJECT/_wiki/wikis/$WIKI_NAME

Published pages:
  [CREATED]  /Architecture          (156 lines, 5 diagrams)
  [UPDATED]  /API-Reference         (203 lines, 3 diagrams)
  [CREATED]  /Data-Model            (178 lines, 2 diagrams)
  [SKIPPED]  /Services              (status: template — run /wiki update services first)
  [CREATED]  /Testing               (145 lines, 3 diagrams)
  [CREATED]  /Deployment            (167 lines, 4 diagrams)
  [CREATED]  /Configuration         (134 lines, 2 diagrams)
  [CREATED]  /Conventions           (189 lines, 2 diagrams)

Result: 6 created | 1 updated | 1 skipped
==========================================
```

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| Wiki not found | No project wiki exists | Auto-create with Step 2 |
| 401 Unauthorized | PAT expired or missing permissions | Check PAT has Wiki Read/Write |
| 409 Conflict | Page was modified remotely | Fetch latest etag, retry update |
| Page path conflict | Path exists as folder/page mismatch | Rename and retry |

If any page fails to publish, continue with remaining pages and report failures at the end.
