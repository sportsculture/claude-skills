# Claude Code Skills

A collection of reusable skills for [Claude Code](https://claude.ai/claude-code) that automate complex workflows with human-in-the-loop decision points.

## What are Skills?

Skills are executable runbooks that Claude Code follows step-by-step. They combine:
- **Automation** - AI handles repetitive tasks (image generation, evaluation, file creation)
- **Human judgment** - Users make taste decisions at key points
- **Implementation output** - Produces ready-to-use code artifacts

## Available Skills

### Productivity

| Skill | Description |
|-------|-------------|
| [`/worklog`](skills/worklog.md) | Generate session worklog and save to Obsidian |
| [`/screenshot`](skills/screenshot.md) | Capture localhost pages for visual verification |
| [`/deploy`](skills/deploy.md) | Deploy application to production (template) |

### Design

| Skill | Description |
|-------|-------------|
| [`/design-explore`](skills/design-explore.md) | AI-powered design exploration with multi-model evaluation |

### Git Worktrees

| Skill | Description |
|-------|-------------|
| [`/worktree-create`](skills/worktree/worktree-create.md) | Create new git worktree for feature development |
| [`/worktree-status`](skills/worktree/worktree-status.md) | Show status of all worktrees |
| [`/worktree-sync-main`](skills/worktree/worktree-sync-main.md) | Sync worktree with main branch |

### DAG Workflow Management

| Skill | Description |
|-------|-------------|
| [`/dag/create-work-dag`](skills/dag/create-work-dag.md) | Create a new work DAG for project planning |
| [`/dag/start-node`](skills/dag/start-node.md) | Start working on a DAG node |
| [`/dag/complete-node`](skills/dag/complete-node.md) | Mark a DAG node as complete |
| [`/dag/show-progress`](skills/dag/show-progress.md) | Display DAG progress overview |
| [`/dag/next-available`](skills/dag/next-available.md) | Find next available node to work on |
| [`/dag/update-node`](skills/dag/update-node.md) | Update node metadata |

---

## Skill Details

### `/worklog` - Session Worklog Generator

Generate a structured worklog from your Claude Code session and save it to an Obsidian vault.

**Workflow:**
1. Detects project from git root or working directory
2. Extracts accomplishments, decisions, files created
3. Creates worklog in Obsidian vault
4. Updates daily note with link to worklog

**Usage:**
```
/worklog
```

**Requirements:**
- Obsidian vault with configured directory structure
- File write permissions in Claude Code

**Cost:** Free (no API calls)

---

### `/screenshot` - Page Screenshot

Take screenshots of localhost pages to verify visual changes (Wayland/COSMIC).

**Usage:**
```
/screenshot [path] [port]
/screenshot --calibrate
```

**Requirements:**
- `grim` + `slurp` (Wayland) or `scrot` (X11)
- `~/bin/screenshot-page` script (see skill for setup)

---

### `/deploy` - Deployment Template

Template skill for deploying applications. Customize for your project.

**Usage:**
```
/deploy
```

Copy to your project and customize build/deploy commands.

---

### `/design-explore` - AI-Powered Design Exploration

Generate visual design directions, get multi-model AI evaluation, and produce implementation-ready CSS/tokens.

**Workflow:**
1. Specify product and aesthetic lens
2. Generate hero images for 4 themes (~$0.56)
3. AI evaluates each direction (Gemini + Claude)
4. Human picks direction or accepts synthesis
5. Outputs CSS variables, design tokens, implementation guide

**Usage:**
```
/design-explore MyApp "retro gaming"
/design-explore FamilyMeetup "military ops console"
```

**Requirements:**
- `GOOGLE_API_KEY` for Nano Banana Pro (gemini-3-pro-image-preview)
- PAL MCP server for multi-model consensus

**Cost:** ~$0.60 per exploration (4 hero images + AI evaluation)

---

### Worktree Skills

Manage git worktrees for parallel feature development.

**Create:** `/worktree-create feat/new-feature`
**Status:** `/worktree-status`
**Sync:** `/worktree-sync-main`

---

### DAG Workflow Skills

Manage work as Directed Acyclic Graphs for complex project planning.

**Create DAG:** `/dag/create-work-dag ecommerce_plan`
**Start work:** `/dag/start-node shopping_cart`
**Complete:** `/dag/complete-node shopping_cart`
**Progress:** `/dag/show-progress`
**Next:** `/dag/next-available`

**Requirements:**
- DAG playground MCP server
- Optional: coding-gps-tui for visual monitoring

---

## Installation

Copy skills to your project's `.claude/commands/` directory:

```bash
# Copy individual skill
mkdir -p .claude/commands
cp skills/worklog.md .claude/commands/

# Copy skill directory (e.g., DAG skills)
cp -r skills/dag .claude/commands/
```

Then invoke with `/skillname` in Claude Code.

## Creating New Skills

Skills are markdown files with:
1. **YAML frontmatter** - `description` field for skill discovery
2. **Header** - Name and description
3. **Usage** - How to invoke the skill
4. **Prerequisites** - Required API keys, tools
5. **Execution stages** - Step-by-step instructions Claude follows
6. **Error handling** - What to do when things fail

Example frontmatter:
```yaml
---
description: Brief description shown in skill list
---
```

See `skills/worklog.md` for a complete example.

## Philosophy

**Human-in-the-loop AI automation:**
- Automate what can be automated
- Pause for human judgment on taste decisions
- Output implementation-ready artifacts
- Track costs transparently

## Directory Structure

```
claude-skills/
├── README.md
└── skills/
    ├── worklog.md           # Obsidian worklog generator
    ├── screenshot.md        # Page screenshot capture
    ├── deploy.md            # Deployment template
    ├── design-explore.md    # AI design exploration
    ├── worktree/            # Git worktree management
    │   ├── worktree-create.md
    │   ├── worktree-status.md
    │   └── worktree-sync-main.md
    └── dag/                 # DAG workflow management
        ├── create-work-dag.md
        ├── start-node.md
        ├── complete-node.md
        ├── show-progress.md
        ├── next-available.md
        └── update-node.md
```

## License

MIT
