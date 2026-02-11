---
description: Organize GitLab repos into subgroups, generate READMEs, and set descriptions
---

# GitLab Group Management

Organize a GitLab group by creating subgroups, transferring repos, generating READMEs for repos missing them, and setting scannable descriptions on all repos.

Requires `glab` CLI authenticated with a token that has `api` scope.

## Usage

```
/gitlab-manage <group-path-or-id>
/gitlab-manage todo-software-studios
/gitlab-manage 121438601
```

Optional argument: `$ARGUMENTS` is the GitLab group path or numeric ID. If not provided, ask the user.

## Steps

### 1. Inventory the group

```bash
# Resolve group ID if path was given
glab api "groups/<path-or-id>" | jq '{id, name, web_url}'

# List all projects with key metadata
glab api "groups/<ID>/projects" --paginate | jq -r \
  '.[] | [.id, .name, .path, .description // "(none)", .empty_repo, .archived, .default_branch, (.readme_url != null)] | @tsv'

# List existing subgroups
glab api "groups/<ID>/subgroups" | jq -r '.[] | [.id, .name] | @tsv'
```

Present the inventory to the user as a table. Call out:
- Repos with `deletion_scheduled` suffixes (need renaming)
- Archived repos (need unarchiving before transfer)
- Empty repos (need initial commit for README)
- Repos missing README (`readme_url` is null)
- Repos missing descriptions

### 2. Propose subgroup structure

Based on the repo names and contents, propose a subgroup organization. Ask the user to confirm or adjust. Consider categories like:
- Domain/product groupings (e.g., sports, ai, apps)
- Infrastructure/tooling
- Legacy/archived

Present the proposed structure as a tree and get approval before proceeding.

### 3. Fix repo names

For any repos with `deletion_scheduled` suffixes or other naming issues:

```bash
glab api -X PUT "projects/<ID>" -f "name=<clean-name>" -f "path=<clean-path>"
```

### 4. Unarchive if needed

Archived repos cannot be transferred. Unarchive first:

```bash
glab api -X POST "projects/<ID>/unarchive"
```

### 5. Create subgroups

```bash
glab api -X POST groups \
  -f "name=<name>" \
  -f "path=<path>" \
  -f "parent_id=<parent-group-id>" \
  -f "visibility=private"
```

Record the returned subgroup IDs for transfers.

### 6. Transfer repos

For each repo, transfer to its designated subgroup:

```bash
glab api -X PUT "projects/<project-id>/transfer" -f "namespace=<subgroup-id>"
```

**Important:** GitLab sets up URL redirects automatically. If a transfer returns 403:
- Check if the repo is archived (unarchive first)
- Check if there's a name collision in the target subgroup
- Verify token has Owner-level access

### 7. Generate READMEs

For repos missing READMEs, use **haiku subagents in parallel** to read local source code and generate concise READMEs (30-80 lines each). Group repos into batches of 5-6 per agent.

Each agent should:
- Read package.json, mix.exs, pyproject.toml, or CLAUDE.md to understand the project
- Generate a README with: H1 project name, 1-2 sentence description, tech stack, setup instructions, current status
- Output in delimited format: `===REPO:name===\n[content]\n===END===`

Push READMEs via GitLab API:

```bash
# For repos with existing content (create file)
glab api -X POST "projects/<ID>/repository/files/README.md" \
  -f "branch=<default-branch>" \
  -f "content=<readme-content>" \
  -f "commit_message=Add README.md" \
  -f "encoding=text"

# For empty repos (initial commit via commits API)
# Use --input with JSON payload containing actions array
```

If create fails with "already exists", use PUT instead of POST to update.

### 8. Set descriptions

Use **a haiku subagent** to read each repo's README/source and generate a one-line description (50-80 chars). Then apply:

```bash
glab api -X PUT "projects/<ID>" -f "description=<description>"
```

### 9. Verify

```bash
# List each subgroup's projects with descriptions
for subgroup_id in <list>; do
  glab api "groups/$subgroup_id/projects" | jq -r '.[] | "\(.name) | \(.description)"'
done

# Verify root group only has expected repos
glab api "groups/<parent-id>/projects" | jq -r '.[].name'
```

Present final summary:
- Subgroup counts (should sum to total migrated)
- README coverage
- Description coverage
- Any repos that couldn't be transferred (with reason)

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 403 on transfer | Repo is archived | `POST projects/<id>/unarchive` first |
| 403 on transfer | Token lacks Owner access | Check group membership |
| 400 on file create | File already exists | Use PUT instead of POST |
| Empty jq output from glab | glab pipes empty to stdout | Save to temp file first, then parse |

## Cost

Free (no AI API calls beyond subagent README/description generation, which uses Claude context).

## Tips

- Run transfers in parallel (8+ concurrent glab calls work fine)
- GitLab rate limits are generous for API mutations (~600/min)
- `glab api` piping to python via stdin can fail silently — save to temp files when parsing complex JSON
- Empty repos need the commits API (not files API) for the initial commit
