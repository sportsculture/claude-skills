---
description: Generate a worklog from this session and save to Obsidian vault
---

# Generate Worklog

Generate a worklog summarizing work done in this Claude Code session and save it to an Obsidian vault.

## Usage

```
/worklog
```

## What This Skill Does

This skill creates a structured worklog from the current conversation:

1. **Detects Project** - Identifies current project from git root or working directory
2. **Summarizes Work** - Extracts accomplishments, decisions, and next steps
3. **Creates Worklog** - Saves to Obsidian vault worklogs folder
4. **Updates Daily Note** - Links worklog from today's daily note (creates if needed)

## Configuration

Set these environment variables or edit the paths in the skill:

```bash
# Obsidian vault paths (defaults shown)
OBSIDIAN_VAULT_ROOT=~/3-Resources/personal-notes
WORKLOG_DIR="2 - Areas/TimeLogs/Worklogs"
DAILY_DIR="2 - Areas/TimeLogs/Daily"
```

## Workflow

When you run `/worklog`, Claude will:

### Step 1: Gather Information

Analyze the current conversation and extract:
- **Project name** from git root or working directory
- **Summary** (2-3 sentences of what was accomplished)
- **Accomplishments** (bullet list of completed work)
- **Decisions** (key choices made and rationale)
- **Files/Scripts created or modified**
- **Next steps** (if any)

### Step 2: Generate Worklog

Create a markdown file with YAML frontmatter:

```yaml
---
date: YYYY-MM-DD
project: <project-name>
tags: [worklog, <project-name>]
repo: <git-remote-url>
branch: <current-branch>
---
```

Content sections:
- **Summary** - Brief overview of the session
- **Work Completed** - Detailed accomplishments with context
- **Scripts/Files Created** - Table of new artifacts
- **Decisions** - Key choices and rationale
- **Next Steps** - Follow-up tasks

### Step 3: Update Daily Note

1. Check if today's daily note exists at `DAILY_DIR/YYYYMMDD.md`

2. If not, create it from template:
   ```markdown
   ---
   creation date: YYYY-MM-DD
   ---

   ## 💭 Quote of the Day

   It's not that I'm so smart, it's just that I stay with problems longer. — Albert Einstein

   ---

   ## 🌙 Dreams

   **Dream:**

   **Pay Attention To / Daily Reflection:**

   ---

   ## 📓 Notes

   ---

   ## 🔨 Work

   ### Worklogs

   ---
   ```

3. Add link to worklog under "### Worklogs" section:
   ```markdown
   - [[<worklog-filename>|<Project> - <Brief Description>]]
   ```

## File Naming Convention

Worklogs: `<project>-YYYY-MM-DD.md`
- Example: `gridplay-2026-01-28.md`

If multiple worklogs for same project/day, append time:
- Example: `gridplay-2026-01-28-1430.md`

## Implementation Steps

When this skill is invoked, Claude should:

1. **Get project info:**
   ```bash
   git rev-parse --show-toplevel 2>/dev/null || pwd
   git remote get-url origin 2>/dev/null || echo ""
   git branch --show-current 2>/dev/null || echo "main"
   ```

2. **Extract from conversation:**
   - Scan conversation for completed tasks, file operations, decisions
   - Group by: accomplishments, decisions, scripts/files, next steps

3. **Check for existing worklog:**
   ```bash
   ls "$OBSIDIAN_VAULT_ROOT/$WORKLOG_DIR/<project>-$(date +%Y-%m-%d)"*.md 2>/dev/null
   ```
   - If exists, consider appending or creating timestamped version

4. **Write worklog file** using the Write tool

5. **Check/create daily note:**
   ```bash
   ls "$OBSIDIAN_VAULT_ROOT/$DAILY_DIR/$(date +%Y%m%d).md" 2>/dev/null
   ```
   - Create from template if missing
   - Add worklog link under "### Worklogs" if not already present

6. **Confirm to user:**
   - Show worklog path
   - Show daily note path
   - Summary of what was logged

## Example Output

After running `/worklog`, Claude confirms:

```
✓ Worklog created: ~/3-Resources/personal-notes/2 - Areas/TimeLogs/Worklogs/gridplay-2026-01-28.md

✓ Daily note updated: ~/3-Resources/personal-notes/2 - Areas/TimeLogs/Daily/20260128.md
  Added link: [[gridplay-2026-01-28|GridPlay - Team Name Sync]]

Summary logged:
- Synced 37 team names in database
- Updated Family Ties sign-in sheet (6 tabs, 97 cells)
- Created 2 reusable sync scripts
```

## Tips

- Run `/worklog` at the end of a work session to capture everything
- The skill reads the full conversation, so no need to summarize first
- Worklogs are linked bidirectionally - daily note links to worklog
- Works from any project on the machine (uses git detection)

## Customization

### Different Vault Structure

Edit the paths in your local copy:
```bash
OBSIDIAN_VAULT_ROOT=~/Documents/MyVault
WORKLOG_DIR="Work/Logs"
DAILY_DIR="Daily"
```

### Different Daily Note Template

Modify the template in Step 3 to match your existing daily note structure.

### Per-Project Overrides

Create `.worklogrc.json` in project root (optional):
```json
{
  "projectName": "My Custom Name",
  "tags": ["client-work", "priority"],
  "skipDailyNote": false
}
```

## Troubleshooting

### "Permission denied"
Ensure the Obsidian vault directories exist and are writable.

### "Daily note not found"
The skill will create one from template. If template is missing, a basic structure is used.

### "Worklog already exists"
A timestamped version will be created to avoid overwriting.

### "Not in a git repository"
The skill falls back to using the current directory name as the project name.

## Requirements

- Claude Code with file write permissions
- Obsidian vault with the configured directory structure
- Git (optional, for project detection)
