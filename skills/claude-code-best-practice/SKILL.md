```markdown
---
name: claude-code-best-practice
description: Master Claude Code features, patterns, and agentic workflows using community best practices
triggers:
  - "how do I use Claude Code effectively"
  - "what are Claude Code best practices"
  - "set up subagents in Claude Code"
  - "create a slash command for Claude"
  - "configure CLAUDE.md memory"
  - "orchestrate agents with Claude Code"
  - "use skills in Claude Code"
  - "vibe coding with Claude"
---

# Claude Code Best Practice

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Community-curated best practices and patterns for [Claude Code](https://code.claude.com) — Anthropic's agentic coding tool. Covers subagents, commands, skills, memory, hooks, MCP servers, orchestration workflows, and advanced features like Auto Mode, Agent Teams, and Scheduled Tasks.

---

## What Claude Code Is

Claude Code is an AI coding agent that runs in your terminal (and web/IDE). It can read, write, and execute code; browse the web; call tools; and spawn subagents. Configuration lives in `.claude/` inside your project.

```
.claude/
├── settings.json       # permissions, model, output style
├── agents/             # subagent definitions
├── commands/           # slash command templates
├── skills/             # skill bundles (SKILL.md)
├── hooks/              # event-driven scripts
└── rules/              # scoped memory rules
CLAUDE.md               # root memory file (always loaded)
.mcp.json               # MCP server connections
```

---

## Installation & Setup

```bash
# Install Claude Code CLI
npm install -g @anthropic/claude-code

# Authenticate
claude auth login

# Launch interactive session
claude

# Launch with auto-permissions (no prompts)
claude --permission-mode auto

# Launch headless (pipe a prompt)
claude -p "Refactor src/utils.ts to use async/await"
```

---

## Core Concepts

### Memory — `CLAUDE.md`

The root memory file is always injected into context. Use it to define project conventions, tech stack, and rules.

```markdown
# CLAUDE.md

## Project
TypeScript monorepo. Node 20. pnpm workspaces.

## Rules
- Never commit directly to `main`
- All new files need a unit test
- Use `zod` for runtime validation

## @imports
@.claude/rules/typescript.md
@.claude/rules/testing.md
```

Import scoped rule files with `@path`:

```bash
# .claude/rules/typescript.md
- Prefer `type` over `interface` for unions
- Enable `strict: true` in tsconfig
```

Auto memory writes: Claude can append learnings to `CLAUDE.md` automatically when you ask it to "remember this".

---

### Subagents — `.claude/agents/<name>.md`

Subagents run in a **fresh isolated context** with their own tools, permissions, model, and identity.

```markdown
<!-- .claude/agents/code-reviewer.md -->
---
name: code-reviewer
description: Reviews PRs for bugs, style issues, and security vulnerabilities
model: claude-opus-4-5
tools:
  - read_file
  - search_code
permissions:
  - read
---

You are a senior code reviewer. When invoked:
1. Read the diff provided
2. Check for security issues (injections, secrets, unvalidated input)
3. Flag style violations against CLAUDE.md rules
4. Suggest improvements with code examples
5. Return a structured review with LGTM / REQUEST_CHANGES verdict
```

Invoke a subagent from a command or prompt:

```
/review-pr diff: <paste diff here>
```

Or spawn programmatically in headless mode:

```bash
claude -p "Use the code-reviewer agent to review the changes in the last commit" \
  --agent code-reviewer
```

---

### Commands — `.claude/commands/<name>.md`

Commands inject a **prompt template** into the current context. They're simple, fast, and user-invoked with `/`.

```markdown
<!-- .claude/commands/fix-types.md -->
---
description: Fix all TypeScript type errors in the current file
---

Look at the currently open file. Run `tsc --noEmit` scoped to this file.
For each type error:
1. Explain what's wrong in one sentence
2. Apply the minimal fix
3. Re-run tsc to confirm it's resolved

Do not change logic — only fix types.
```

Usage in Claude Code:

```
/fix-types
```

---

### Skills — `.claude/skills/<name>/SKILL.md`

Skills are **discoverable knowledge bundles** with frontmatter triggers. Claude auto-loads them when a trigger phrase is detected.

```
.claude/skills/
└── openapi-codegen/
    └── SKILL.md
```

```markdown
<!-- .claude/skills/openapi-codegen/SKILL.md -->
---
name: openapi-codegen
description: Generate TypeScript clients from OpenAPI specs
triggers:
  - "generate API client"
  - "create types from OpenAPI"
  - "scaffold from swagger"
---

# OpenAPI Codegen Skill

Use `openapi-typescript` to generate types:

```bash
npx openapi-typescript ./openapi.yaml -o src/types/api.d.ts
```

Then use `openapi-fetch` for type-safe calls:

```typescript
import createClient from "openapi-fetch";
import type { paths } from "./types/api.d.ts";

const client = createClient<paths>({ baseUrl: process.env.API_BASE_URL });

const { data, error } = await client.GET("/users/{id}", {
  params: { path: { id: "123" } },
});
```
```

---

### Settings — `.claude/settings.json`

```json
{
  "model": "claude-opus-4-5",
  "permissionMode": "auto",
  "outputStyle": "minimal",
  "fastMode": true,
  "statusLine": {
    "show": true,
    "items": ["context", "model", "cost", "session"]
  },
  "permissions": {
    "allow": ["read", "write", "execute"],
    "deny": ["network:external"]
  },
  "keybindings": {
    "voice": "ctrl+shift+v"
  }
}
```

Global settings live in `~/.claude/settings.json`. Project settings override globals.

---

### MCP Servers — `.mcp.json`

Connect external tools, databases, and APIs via the Model Context Protocol.

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    }
  }
}
```

---

### Hooks — `.claude/hooks/`

Hooks are scripts that fire on Claude Code events **outside the agentic loop**.

```json
// .claude/settings.json
{
  "hooks": {
    "postToolUse": ".claude/hooks/post-tool.sh",
    "onSessionEnd": ".claude/hooks/summarize.sh",
    "preFileWrite": ".claude/hooks/lint-check.sh"
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/lint-check.sh
# Runs before Claude writes a file — blocks if lint fails

FILE=$CLAUDE_HOOK_FILE
npx eslint "$FILE" --max-warnings 0
exit $?
```

---

## Orchestration Workflow Pattern

The recommended pattern: **Command → Agent → Skill**

```
User types /weather-orchestrator city=London
        │
        ▼
┌─────────────────┐
│    Command      │  Parses args, sets up context
│ weather-        │  Invokes subagent with task
│ orchestrator.md │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Subagent     │  Isolated context, specialized model
│ weather-        │  Uses skills for domain knowledge
│ fetcher.md      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Skill       │  Auto-loaded by trigger phrase
│ weather-api/    │  Provides API patterns + examples
│ SKILL.md        │
└─────────────────┘
```

```markdown
<!-- .claude/commands/weather-orchestrator.md -->
---
description: Fetch and display weather for a city
args:
  city: string
---

Fetch weather for {{city}}.
Use the weather-fetcher subagent.
Format the result as a concise summary with temperature, conditions, and 3-day forecast.
```

---

## Advanced Features

### Agent Teams (parallel agents)

```bash
# Enable via env var
export CLAUDE_AGENT_TEAMS=true
claude
```

```
/team-task
  agent-1: Refactor the auth module
  agent-2: Write tests for the auth module
  agent-3: Update the auth documentation
```

Agents share task coordination. Results are merged by the orchestrating session.

### Scheduled Tasks

```
# Run locally on a schedule (up to 3 days)
/loop every 30m: check for new GitHub issues and triage them

# Run in Anthropic cloud (works when machine is off)
/schedule daily at 9am: generate a changelog from commits since yesterday
```

### Auto Mode

```bash
claude --permission-mode auto
```

Background safety classifier handles all permission decisions. No manual prompts. Blocks prompt injection and risky escalations automatically.

### Git Worktrees (parallel branches)

```bash
# Each agent gets an isolated working copy
git worktree add ../feature-auth -b feature/auth
git worktree add ../feature-payments -b feature/payments

# Run Claude in each worktree simultaneously
cd ../feature-auth && claude -p "Implement JWT auth"
cd ../feature-payments && claude -p "Integrate Stripe"
```

### Voice Dictation

```
/voice
# Press Ctrl+Shift+V (rebindable), speak your prompt, release to send
# Supports 20 languages
```

### Remote Control

```
/remote-control
# or
/rc
```

Generates a URL to continue the session from any device (phone, tablet, browser).

---

## Common Patterns

### Pattern 1: TDD Loop

```markdown
<!-- .claude/commands/tdd.md -->
Write a failing test for {{feature}} first.
Run the tests to confirm failure.
Implement the minimal code to pass.
Refactor. Run tests again to confirm green.
Commit with message: "feat: {{feature}}"
```

### Pattern 2: PR Review Pipeline

```bash
# Headless PR review
git diff main...HEAD | claude -p "Review this diff for bugs and security issues" \
  --agent code-reviewer \
  --output-format json > review.json
```

### Pattern 3: Batch File Operations

```
/batch across src/**/*.ts: add JSDoc comments to all exported functions
```

### Pattern 4: Ralph Wiggum Loop (long-running tasks)

```bash
# Install the plugin
claude plugin install ralph-wiggum

# Run autonomous development loop
/ralph-wiggum "Build a REST API for user management with auth, CRUD, and tests"
# Iterates until the task is complete
```

---

## CLI Reference

```bash
# Interactive
claude                                    # start session
claude --model claude-opus-4-5            # specify model
claude --permission-mode auto             # no permission prompts
claude --permission-mode full             # allow everything
claude --fast                             # fast mode (reduced quality, lower cost)
claude --channels telegram                # enable Telegram channel

# Headless
claude -p "prompt"                        # one-shot prompt
claude -p "prompt" --output-format json   # JSON output
claude -p "prompt" --no-stream           # wait for full response

# Session management
claude --continue                         # resume last session
claude --session <id>                     # resume specific session
/rewind                                   # undo last file edits (or Esc Esc)

# Utilities
claude auth login                         # authenticate
claude config list                        # show active config
claude plugin install <name>              # install a plugin
claude mcp list                           # list MCP servers
```

---

## Troubleshooting

**Claude ignores CLAUDE.md rules**
- Check that `@imports` paths are relative and files exist
- Avoid rules files > 2000 tokens — split them
- Prefix critical rules with `IMPORTANT:` or `ALWAYS:`

**Subagent not found**
- File must be at `.claude/agents/<name>.md` (exact match, kebab-case)
- YAML frontmatter `name:` must match the filename

**MCP server won't connect**
- Run `claude mcp list` to see status
- Check env vars are exported in your shell before running `claude`
- Use absolute paths in `args` if relative paths fail

**Hooks blocking writes unexpectedly**
- Hook exit code `0` = allow, non-zero = block
- Test hooks manually: `bash .claude/hooks/lint-check.sh`
- Check `CLAUDE_HOOK_FILE` env var is set when testing

**Context window filling up**
- Use `/simplify` to refactor verbose code before continuing
- Split tasks across subagents (each gets a fresh context)
- Use `@path` imports in CLAUDE.md to load rules lazily

**Scheduled tasks not running**
- `/loop` requires the CLI to stay running; use `/schedule` for cloud execution
- Check `claude auth login` is current — scheduled tasks use stored credentials

---

## Resources

- [Claude Code Docs](https://code.claude.com/docs/en)
- [Official Skills Repo](https://github.com/anthropics/skills)
- [Prompt Engineering Tutorial](https://github.com/anthropics/prompt-eng-interactive-tutorial)
- [Claude Code Hooks](https://github.com/shanraisshan/claude-code-hooks)
- [Boris Cherny on X](https://x.com/bcherny)
```
