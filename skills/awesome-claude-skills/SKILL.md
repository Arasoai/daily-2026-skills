```markdown
---
name: awesome-claude-skills
description: Curated collection of Claude Skills for Claude.ai, Claude Code, and Claude API — plus Composio integration for connecting Claude to 500+ apps
triggers:
  - add a claude skill to my project
  - connect claude to external apps
  - install a claude skill
  - create a CLAUDE.md skill
  - use composio with claude
  - set up claude code plugin
  - automate workflows with claude skills
  - find skills for claude code
---

# Awesome Claude Skills

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated collection of Claude Skills that extend Claude's capabilities across Claude.ai, Claude Code, and the Claude API. Skills teach Claude repeatable, standardized workflows — from document processing and code tools to connecting Claude to 500+ external apps via Composio.

---

## What Are Claude Skills?

Claude Skills are markdown files (typically `SKILL.md` or placed in `.claude/skills/`) that give Claude specialized knowledge and workflows. They are installed into Claude Code as plugins or referenced via `CLAUDE.md`.

Skills enable Claude to:
- Follow consistent, repeatable workflows
- Integrate with external services (GitHub, Slack, Gmail, Notion, etc.)
- Apply domain-specific expertise (security, architecture, testing)
- Automate multi-step tasks autonomously

---

## Installation

### Install a Skill into Claude Code

```bash
# From a local directory
claude --plugin-dir ./my-skill-plugin

# Install Composio connect-apps plugin (most popular)
git clone https://github.com/ComposioHQ/awesome-claude-skills.git
cd awesome-claude-skills
claude --plugin-dir ./connect-apps-plugin
```

### Add a Skill to Your Project

Skills referenced in `CLAUDE.md` are automatically picked up by Claude Code:

```bash
# Create a skills directory in your project
mkdir -p .claude/skills

# Copy or write a skill file
cp path/to/SKILL.md .claude/skills/my-skill.md
```

Reference it in your `CLAUDE.md`:

```markdown
## Skills

@.claude/skills/my-skill.md
```

---

## Connecting Claude to 500+ Apps (Composio)

The `connect` skill lets Claude take real actions — send emails, create GitHub issues, post Slack messages, update Notion databases.

### Setup

```bash
# 1. Install the plugin
claude --plugin-dir ./connect-apps-plugin

# 2. Run the setup wizard inside Claude Code
/connect-apps:setup

# 3. Paste your API key when prompted
# Get a free key at: https://platform.composio.dev
```

Set your API key as an environment variable:

```bash
export COMPOSIO_API_KEY="your_api_key_here"
```

### Usage Examples

Once connected, ask Claude naturally:

```
Send an email to john@example.com summarizing today's standup notes
```

```
Create a GitHub issue in my-org/my-repo titled "Fix login bug" with steps to reproduce
```

```
Post a message to the #deployments Slack channel: "v2.3.1 deployed to production"
```

```
Add a new page to my Notion database with today's meeting notes
```

### Python: Using Composio Programmatically

```python
import os
from composio import ComposioToolSet, App

toolset = ComposioToolSet(api_key=os.environ["COMPOSIO_API_KEY"])

# Get tools for specific apps
gmail_tools = toolset.get_tools(apps=[App.GMAIL])
github_tools = toolset.get_tools(apps=[App.GITHUB])
slack_tools = toolset.get_tools(apps=[App.SLACK])

# Use with an LLM (e.g., Anthropic)
import anthropic

client = anthropic.Anthropic()
all_tools = toolset.get_tools(apps=[App.GMAIL, App.GITHUB, App.SLACK])

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=1024,
    tools=all_tools,
    messages=[
        {
            "role": "user",
            "content": "Create a GitHub issue for the bug we discussed and notify the team on Slack"
        }
    ]
)

# Handle tool calls
result = toolset.handle_tool_calls(response)
print(result)
```

---

## Key Skills Reference

### Document Processing

| Skill | What it Does |
|-------|-------------|
| `docx` | Create/edit/analyze Word docs with tracked changes |
| `pdf` | Extract text, tables, metadata; merge & annotate PDFs |
| `pptx` | Read, generate, and adjust PowerPoint slides |
| `xlsx` | Spreadsheet manipulation: formulas, charts, transformations |

```bash
# Install official Anthropic document skills
git clone https://github.com/anthropics/skills.git
ls skills/skills/  # docx, pdf, pptx, xlsx
```

### Development & Code Tools

```bash
# Changelog generator — transforms git commits to user-facing changelogs
# Place SKILL.md in .claude/skills/changelog-generator.md
# Then ask Claude: "Generate a changelog from the last 10 commits"

# MCP Builder — scaffold MCP servers
# Ask: "Create an MCP server for the Stripe API"

# Webapp Testing with Playwright
# Ask: "Test the login flow on localhost:3000 and take screenshots"
```

### Data & Analysis

```python
# postgres skill — safe read-only queries
# Configure your connection string:
export POSTGRES_CONNECTION_STRING="postgresql://user:pass@localhost:5432/mydb"

# Then ask Claude: "Show me the top 10 customers by revenue this month"
```

---

## Creating Your Own Skill

### Minimal Skill Structure

```markdown
---
name: my-skill-name
description: One-line description of what this skill does
triggers:
  - phrase users might say to invoke this
  - another natural trigger phrase
  - do the thing this skill handles
---

# My Skill Name

## Overview
What this skill does and when to use it.

## Instructions
Step-by-step guidance for Claude to follow.

## Examples
Concrete examples of inputs and expected outputs.
```

### Using the Skill Creator

Install the built-in skill creator skill, then ask:

```
Create a skill that [describes your workflow]
```

Claude will scaffold a complete `SKILL.md` with frontmatter, instructions, and examples.

### Skill Creator Python Helper

```python
# skill_creator.py — programmatically generate skill stubs
import anthropic
import yaml

def create_skill(name: str, description: str, workflow: str) -> str:
    client = anthropic.Anthropic()
    
    prompt = f"""Create a Claude Skill SKILL.md file for the following:
    
Name: {name}
Description: {description}
Workflow: {workflow}

Include YAML frontmatter with name, description, and 6 triggers.
Include sections: Overview, When to Use, Instructions, Examples, Configuration."""

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )
    
    return response.content[0].text

skill_content = create_skill(
    name="api-rate-limit-handler",
    description="Automatically handles API rate limiting with exponential backoff",
    workflow="Detect 429 responses, implement backoff, retry requests, log outcomes"
)

with open("SKILL.md", "w") as f:
    f.write(skill_content)
```

---

## Skill Directory Structure

```
your-project/
├── CLAUDE.md                    # References skills with @path syntax
├── .claude/
│   └── skills/
│       ├── testing.md           # TDD workflow skill
│       ├── git-workflow.md      # Git branching/PR skill
│       └── api-integration.md  # API patterns skill
└── connect-apps-plugin/         # Composio plugin directory
    ├── plugin.json
    └── commands/
        └── setup.js
```

### CLAUDE.md Integration Pattern

```markdown
# Project: My Application

## Active Skills

@.claude/skills/testing.md
@.claude/skills/git-workflow.md

## Project Context
This is a Python FastAPI application with PostgreSQL.
```

---

## Popular Community Skills

```bash
# iOS Simulator testing
git clone https://github.com/conorluddy/ios-simulator-skill

# D3.js data visualizations
git clone https://github.com/chrisvoncsefalvay/claude-d3js-skill

# Playwright browser automation
git clone https://github.com/lackeyjb/playwright-skill

# AWS CDK best practices
git clone https://github.com/zxkane/aws-skills

# EPUB generation from markdown
git clone https://github.com/smerchek/claude-epub-skill

# Install any skill by adding SKILL.md to .claude/skills/
cp claude-d3js-skill/SKILL.md .claude/skills/d3-visualization.md
```

---

## Common Patterns

### Delegating to Sub-Agents

```markdown
# In your skill file
## Sub-Agent Pattern

When the task involves parallel work:
1. Identify independent subtasks
2. Dispatch separate agents for each using the Task tool
3. Collect results and synthesize
4. Run a final review pass
```

### Test-Driven Development Skill Usage

```
# Trigger the TDD skill
"Implement user authentication using TDD"

# Claude will:
# 1. Write failing tests first
# 2. Implement minimal passing code
# 3. Refactor while keeping tests green
# 4. Document the implementation
```

### Git Worktrees for Parallel Features

```bash
# Using the git-worktrees skill
# Ask: "Set up a worktree for the feature/payment-integration branch"

# Claude will run:
git worktree add ../project-payment feature/payment-integration
cd ../project-payment
# Work in isolation without affecting main branch
```

---

## Troubleshooting

### Skill Not Being Recognized

```bash
# Verify Claude Code can see your skill
cat CLAUDE.md  # Should have @path references

# Check skill file syntax
head -20 .claude/skills/my-skill.md  # Should start with --- YAML frontmatter

# Restart Claude Code after adding new skills
exit
claude
```

### Composio Connection Issues

```bash
# Verify API key is set
echo $COMPOSIO_API_KEY

# Re-run setup
/connect-apps:setup

# Check connected apps
/connect-apps:list

# Test a specific connection
/connect-apps:test gmail
```

### Skill Triggers Not Firing

```yaml
# Make triggers natural and varied in your frontmatter
triggers:
  - generate a changelog          # imperative
  - create release notes          # synonym
  - what changed in this release  # question form
  - summarize git commits         # descriptive
  - prepare a changelog from git  # verbose form
  - update the changelog file     # action-oriented
```

### WebFetch Blocked (Use reddit-fetch Pattern)

```bash
# When direct fetch returns 403, use the reddit-fetch skill pattern
# It routes through Gemini CLI as a fallback
# Ask: "Fetch the content from [url] using the reddit-fetch approach"
```

---

## Resources

- **Official Skills Repository**: https://github.com/anthropics/skills
- **Awesome Claude Skills**: https://github.com/ComposioHQ/awesome-claude-skills
- **Composio Platform**: https://platform.composio.dev
- **Supported Apps (1000+)**: https://composio.dev/toolkits
- **Claude Code Docs**: https://docs.anthropic.com/en/docs/claude-code
- **MCP Protocol**: https://modelcontextprotocol.io
- **Discord Community**: https://discord.com/invite/composio

---

## Contributing a Skill

```bash
# Fork the repository
git clone https://github.com/ComposioHQ/awesome-claude-skills.git
cd awesome-claude-skills

# Create your skill directory
mkdir skills/my-awesome-skill
cat > skills/my-awesome-skill/SKILL.md << 'EOF'
---
name: my-awesome-skill
description: What your skill does in one line
triggers:
  - natural phrase 1
  - natural phrase 2
  - natural phrase 3
  - natural phrase 4
  - natural phrase 5
  - natural phrase 6
---

# My Awesome Skill

## Overview
...
EOF

# Submit a pull request
git checkout -b add-my-awesome-skill
git add skills/my-awesome-skill/
git commit -m "Add my-awesome-skill: brief description"
git push origin add-my-awesome-skill
# Open PR at github.com/ComposioHQ/awesome-claude-skills
```
```
