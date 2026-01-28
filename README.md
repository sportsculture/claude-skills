# Claude Code Skills

A collection of reusable skills for [Claude Code](https://claude.ai/claude-code) that automate complex workflows with human-in-the-loop decision points.

## What are Skills?

Skills are executable runbooks that Claude Code follows step-by-step. They combine:
- **Automation** - AI handles repetitive tasks (image generation, evaluation, file creation)
- **Human judgment** - Users make taste decisions at key points
- **Implementation output** - Produces ready-to-use code artifacts

## Available Skills

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

## Installation

Copy skills to your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp skills/design-explore.md .claude/skills/
```

Then invoke with `/design-explore` in Claude Code.

## Creating New Skills

Skills are markdown files with:
1. **Header** - Name and description
2. **Invocation** - How to call it
3. **Prerequisites** - Required API keys, tools
4. **Execution stages** - Step-by-step instructions Claude follows
5. **Error handling** - What to do when things fail

See `skills/design-explore.md` for a complete example.

## Philosophy

**Human-in-the-loop AI automation:**
- Automate what can be automated
- Pause for human judgment on taste decisions
- Output implementation-ready artifacts
- Track costs transparently

## License

MIT
