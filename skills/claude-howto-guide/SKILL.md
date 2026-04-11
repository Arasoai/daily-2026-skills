```markdown
---
name: claude-howto-guide
description: Master Claude Code features — slash commands, memory, hooks, subagents, MCP, skills, plugins, checkpoints, and CLI — using the claude-howto structured tutorial guide.
triggers:
  - how do I use Claude Code effectively
  - set up Claude Code slash commands
  - configure hooks in Claude Code
  - create a subagent workflow with Claude Code
  - install MCP servers for Claude Code
  - use Claude Code memory and CLAUDE.md
  - set up a Claude Code plugin
  - automate code review with Claude Code
---

# Claude How-To: Master Claude Code Features

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`claude-howto` is a structured, visual, example-driven tutorial guide for Claude Code. It covers every major feature — slash commands, memory, skills, subagents, MCP, hooks, plugins, checkpoints, and CLI — with copy-paste templates, Mermaid diagrams, and a progressive 11–13 hour learning path.

---

## Installation

```bash
git clone https://github.com/luongnv89/claude-howto.git
cd claude-howto
```

No Python dependencies are required to use the templates. To build the offline EPUB:

```bash
uv run scripts/build_epub.py
```

---

## Repository Structure

```
claude-howto/
├── 01-slash-commands/     # User-invoked shortcuts (/cmd)
├── 02-memory/             # Persistent context (CLAUDE.md)
├── 03-skills/             # Reusable capabilities (auto-invoked)
├── 04-subagents/          # Specialized AI assistants
├── 05-mcp/                # External tool access via MCP protocol
├── 06-hooks/              # Event-driven automation
├── 07-plugins/            # Bundled feature packages
├── 08-checkpoints/        # Session snapshots and rewind
├── 09-advanced-features/  # Planning, thinking, background tasks
├── 10-cli/                # CLI commands, flags, options
├── LEARNING-ROADMAP.md    # Guided learning path
├── CATALOG.md             # Full feature catalog
└── CONTRIBUTING.md
```

---

## Quick 15-Minute Setup

```bash
# Create Claude Code command directory in your project
mkdir -p /path/to/your-project/.claude/commands

# Copy a slash command template
cp 01-slash-commands/optimize.md /path/to/your-project/.claude/commands/

# Set up project memory
cp 02-memory/project-CLAUDE.md /path/to/your-project/CLAUDE.md

# Install a skill
cp -r 03-skills/code-review ~/.claude/skills/
```

---

## Feature 1: Slash Commands

Slash commands are Markdown files in `.claude/commands/`. The filename becomes the command name.

**File:** `.claude/commands/review.md`

```markdown
# Code Review

Review the current file or selection for:
- Logic errors and edge cases
- Performance bottlenecks
- Security vulnerabilities
- Style and readability

Provide a structured report with severity levels (critical / warning / suggestion).
```

**Usage in Claude Code:**
```
/review
```

**Copy all example commands:**
```bash
cp 01-slash-commands/*.md .claude/commands/
```

---

## Feature 2: Memory (CLAUDE.md)

`CLAUDE.md` files give Claude persistent context about your project. They are auto-loaded at session start.

**Scopes:**
- `~/.claude/CLAUDE.md` — global, applies to all projects
- `./CLAUDE.md` — project-level
- `./src/CLAUDE.md` — directory-level

**Template:** `./CLAUDE.md`

```markdown
# Project: my-api

## Stack
- Python 3.12, FastAPI, PostgreSQL
- Tests: pytest, httpx
- Linting: ruff, mypy

## Conventions
- All endpoints return `{"data": ..., "error": null}` or `{"data": null, "error": "..."}`
- Use `async def` for all route handlers
- Database sessions via `get_db()` dependency injection

## Key Commands
- `make test` — run test suite
- `make lint` — ruff + mypy
- `make migrate` — run Alembic migrations

## Do Not
- Never commit secrets or `.env` files
- Never use `print()` for logging — use `structlog`
```

**Copy the template:**
```bash
cp 02-memory/project-CLAUDE.md ./CLAUDE.md
```

---

## Feature 3: Skills

Skills are reusable capability definitions that Claude invokes automatically based on context. They live in `~/.claude/skills/` (global) or `.claude/skills/` (project).

**Structure:**
```
~/.claude/skills/
└── code-review/
    ├── skill.md        # Skill definition
    └── templates/      # Supporting templates
```

**Install a skill:**
```bash
cp -r 03-skills/code-review ~/.claude/skills/
```

**Example `skill.md`:**
```markdown
# Skill: Code Review

Trigger: When reviewing code, PRs, or diffs.

## Behavior
1. Check for security vulnerabilities (injection, secrets, auth bypass)
2. Identify performance issues (N+1 queries, unbounded loops)
3. Verify error handling completeness
4. Assess test coverage gaps
5. Output findings as a structured Markdown report
```

---

## Feature 4: Subagents

Subagents are specialized Claude instances delegated subtasks. Define them in `.claude/agents/`.

**File:** `.claude/agents/security-auditor.md`

```markdown
# Agent: Security Auditor

## Role
Specialized security review agent. Focus exclusively on:
- Injection vulnerabilities (SQL, command, LDAP)
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure dependencies

## Output Format
Return a JSON report:
{
  "critical": [...],
  "high": [...],
  "medium": [...],
  "low": [...]
}
```

**Orchestrating subagents in a workflow:**

```python
# Example: Trigger subagent delegation via Claude Code SDK
import anthropic

client = anthropic.Anthropic()

orchestrator_prompt = """
You are an orchestrator. For the following code diff, delegate to:
1. The security-auditor agent for vulnerability scanning
2. The performance-reviewer agent for bottleneck detection

Return a combined report.

Code diff:
{diff}
""".format(diff=open("changes.diff").read())

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    messages=[{"role": "user", "content": orchestrator_prompt}]
)
print(response.content[0].text)
```

---

## Feature 5: MCP (Model Context Protocol)

MCP servers give Claude access to external tools and live data. Configure in `.claude/mcp.json`.

**File:** `.claude/mcp.json`

```json
{
  "servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/project"]
    }
  }
}
```

**Environment variables (never hardcode):**
```bash
export GITHUB_TOKEN="your-token-here"
export DATABASE_URL="postgresql://user:pass@localhost/db"
```

**Copy MCP config templates:**
```bash
cp 05-mcp/mcp.json .claude/mcp.json
```

---

## Feature 6: Hooks

Hooks are scripts triggered by Claude Code events. They live in `.claude/hooks/`.

**Supported events:**
| Event | Trigger |
|-------|---------|
| `pre-tool-use` | Before Claude runs any tool |
| `post-tool-use` | After a tool completes |
| `pre-file-write` | Before writing a file |
| `post-file-write` | After writing a file |
| `session-start` | When a session begins |
| `session-end` | When a session ends |

**File:** `.claude/hooks/post-file-write.sh`

```bash
#!/bin/bash
# Auto-run linter after Claude writes a Python file

FILE="$1"

if [[ "$FILE" == *.py ]]; then
  echo "Running ruff on $FILE..."
  ruff check --fix "$FILE"
  mypy "$FILE" --ignore-missing-imports
fi
```

**File:** `.claude/hooks/pre-file-write.py`

```python
#!/usr/bin/env python3
"""Block writes to protected paths."""

import sys
import os

PROTECTED = [".env", "secrets.json", "credentials.yaml"]

file_path = sys.argv[1] if len(sys.argv) > 1 else ""
filename = os.path.basename(file_path)

if filename in PROTECTED:
    print(f"BLOCKED: Writing to {filename} is not allowed.", file=sys.stderr)
    sys.exit(1)

sys.exit(0)
```

**Register hooks in `.claude/config.json`:**

```json
{
  "hooks": {
    "post-file-write": ".claude/hooks/post-file-write.sh",
    "pre-file-write": ".claude/hooks/pre-file-write.py"
  }
}
```

**Copy hook templates:**
```bash
cp 06-hooks/*.sh .claude/hooks/
cp 06-hooks/*.py .claude/hooks/
chmod +x .claude/hooks/*
```

---

## Feature 7: Plugins

Plugins bundle slash commands, skills, hooks, and memory into a single installable package.

**Plugin structure:**
```
my-plugin/
├── plugin.json
├── commands/
│   └── deploy.md
├── hooks/
│   └── pre-deploy.sh
├── skills/
│   └── deployment/skill.md
└── memory/
    └── CLAUDE.md
```

**File:** `plugin.json`

```json
{
  "name": "devops-pipeline",
  "version": "1.0.0",
  "description": "Full DevOps automation plugin",
  "commands": ["commands/deploy.md"],
  "hooks": {
    "pre-tool-use": "hooks/pre-deploy.sh"
  },
  "skills": ["skills/deployment/"],
  "memory": "memory/CLAUDE.md"
}
```

**Install a plugin:**
```bash
# Copy from claude-howto examples
cp -r 07-plugins/devops-pipeline .claude/plugins/devops-pipeline
```

---

## Feature 8: Checkpoints

Checkpoints save session state so you can rewind to a known-good point.

**In Claude Code:**
```
/checkpoint save before-refactor
/checkpoint list
/checkpoint restore before-refactor
```

**Use case — safe large refactoring:**
```
1. /checkpoint save pre-auth-rewrite
2. Ask Claude to rewrite authentication module
3. Run tests: make test
4. If tests fail: /checkpoint restore pre-auth-rewrite
```

---

## Feature 9: Advanced Features

### Planning Mode

Force Claude to plan before acting:

```
/plan Refactor the user authentication module to use JWT tokens
```

Or in a slash command:

```markdown
# Refactor Plan

Before making any changes:
1. Analyze the current implementation
2. Identify all affected files and dependencies
3. Write a step-by-step migration plan
4. Ask for approval before proceeding

Then execute the approved plan.
```

### Extended Thinking

For complex problems, enable deep reasoning:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    messages=[{
        "role": "user",
        "content": "Design a distributed rate limiter for 10M req/s with Redis"
    }]
)

for block in response.content:
    if block.type == "thinking":
        print("Reasoning:", block.thinking)
    elif block.type == "text":
        print("Answer:", block.text)
```

### Background Tasks

Long-running tasks that don't block your session:

```
/background Run the full test suite and report results when complete
/background Generate API documentation for all endpoints
/background Scan all Python files for security vulnerabilities
```

---

## Feature 10: CLI Reference

Key Claude Code CLI commands and flags:

```bash
# Start interactive session
claude

# Run a one-shot prompt (non-interactive)
claude -p "Explain the auth middleware in src/middleware/auth.py"

# Run with a specific model
claude --model claude-opus-4-6

# Read-only mode (no file writes)
claude --read-only

# Output as JSON (for scripting)
claude -p "List all TODO comments" --output-format json

# Continue last session
claude --continue

# Resume a specific session
claude --resume <session-id>

# Set system prompt
claude --system-prompt "You are a security-focused reviewer"

# Pipe input
cat src/api.py | claude -p "Review this file for security issues"

# Run in a specific directory
claude --cwd /path/to/project
```

**Scripting with Claude Code CLI:**

```bash
#!/bin/bash
# ci-review.sh — Run Claude Code review in CI pipeline

FILES=$(git diff --name-only HEAD~1 HEAD | grep '\.py$')

for FILE in $FILES; do
  echo "Reviewing $FILE..."
  claude -p "Review $FILE for bugs and security issues. Output JSON." \
    --output-format json \
    --read-only \
    --cwd "$(pwd)" \
    >> review-results.json
done

echo "Review complete. Results in review-results.json"
```

---

## Common Workflow Patterns

### Pattern 1: Automated Code Review Pipeline

```
Slash Command (/review)
  → Subagent: security-auditor
  → Subagent: performance-reviewer
  → Hook (post-tool-use): format and post results
  → Memory: store findings in CLAUDE.md
```

**`.claude/commands/review.md`:**
```markdown
# Full Code Review

Orchestrate a comprehensive review:

1. Delegate security analysis to the security-auditor agent
2. Delegate performance analysis to the performance-reviewer agent  
3. Combine results into a single Markdown report
4. Save a summary to `.claude/review-log.md`

Format: Use severity badges (🔴 Critical, 🟠 High, 🟡 Medium, 🟢 Low)
```

### Pattern 2: Safe Refactoring

```bash
# 1. Save checkpoint
# In Claude Code: /checkpoint save pre-refactor

# 2. Run refactor with planning mode
# /plan Refactor database layer to use async SQLAlchemy

# 3. Validate
make test

# 4. On failure, restore
# /checkpoint restore pre-refactor
```

### Pattern 3: CI/CD Integration

```bash
# .github/workflows/claude-review.yml equivalent as shell

#!/bin/bash
# Run on every PR

PR_DIFF=$(git diff main...HEAD)

echo "$PR_DIFF" | claude \
  -p "Review this PR diff. Return JSON with: summary, risks, suggestions." \
  --output-format json \
  --read-only > pr-review.json

# Post results to PR via GitHub API
curl -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"$(cat pr-review.json | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d[\"summary\"])')\"}" \
  "https://api.github.com/repos/$REPO/issues/$PR_NUMBER/comments"
```

---

## Built-In Self-Assessment

Run directly inside Claude Code:

```
/self-assessment          # Full quiz — identifies your gaps and builds a personalized roadmap
/lesson-quiz hooks        # Quiz on hooks specifically
/lesson-quiz subagents    # Quiz on subagents
/lesson-quiz mcp          # Quiz on MCP protocol
```

---

## Learning Path (Recommended Order)

| Step | Module | Time |
|------|--------|------|
| 1 | Slash Commands | 30 min |
| 2 | Memory | 45 min |
| 3 | Checkpoints | 45 min |
| 4 | CLI Basics | 30 min |
| 5 | Skills | 1 hour |
| 6 | Hooks | 1 hour |
| 7 | MCP | 1 hour |
| 8 | Subagents | 1.5 hours |
| 9 | Advanced Features | 2–3 hours |
| 10 | Plugins | 2 hours |

---

## Troubleshooting

**Slash command not appearing:**
- Verify the file is in `.claude/commands/` (project) or `~/.claude/commands/` (global)
- Filename must end in `.md`
- Restart Claude Code session

**CLAUDE.md not loading:**
- Must be named exactly `CLAUDE.md` (case-sensitive)
- Place in project root or relevant subdirectory
- Check for YAML syntax errors if using frontmatter

**Hook not triggering:**
- Ensure hook script is executable: `chmod +x .claude/hooks/myhook.sh`
- Verify hook is registered in `.claude/config.json`
- Check hook output for errors: hooks that exit non-zero block the action

**MCP server not connecting:**
- Confirm environment variables are exported: `echo $GITHUB_TOKEN`
- Test the MCP server binary directly: `npx -y @modelcontextprotocol/server-github`
- Check `.claude/mcp.json` for JSON syntax errors

**Subagent not delegating:**
- Agent definition file must be in `.claude/agents/`
- Verify the orchestrator prompt explicitly references the agent by name
- Check that the agent `.md` file has a clear `## Role` section

---

## Resources

- **Repo:** https://github.com/luongnv89/claude-howto
- **Learning Roadmap:** `LEARNING-ROADMAP.md`
- **Feature Catalog:** `CATALOG.md`
- **Claude Code Docs:** https://code.claude.com
- **Contributing:** `CONTRIBUTING.md`
- **License:** MIT
```
