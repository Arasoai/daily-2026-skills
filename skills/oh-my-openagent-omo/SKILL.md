```markdown
---
name: oh-my-openagent-omo
description: Expert skill for oh-my-openagent (omo) — the multi-model AI agent harness built on OpenCode with ultrawork orchestration, discipline agents, and multi-provider support
triggers:
  - "set up oh-my-openagent"
  - "configure omo ultrawork"
  - "install oh-my-opencode"
  - "use ultrawork with claude"
  - "set up multi-model agent orchestration"
  - "configure omo with openai and anthropic"
  - "run discipline agents with omo"
  - "get oh-my-openagent working"
---

# Oh My OpenAgent (omo) — AI Agent Harness

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

oh-my-openagent (omo) is an open-source agent harness built on top of OpenCode that lets you orchestrate multiple AI models (Claude, GPT, Gemini, Kimi, GLM, Minimax, etc.) in parallel. It ships with pre-configured discipline agents, the `ultrawork` command, hash-anchored editing, LSP + AST-grep tooling, built-in MCPs, and a tmux-integrated TUI.

---

## Installation

### Via AI Agent (recommended)

Paste this into Claude Code, Cursor, AmpCode, or any coding agent:

```
Install and configure oh-my-opencode by following the instructions here:
https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/refs/heads/dev/docs/guide/installation.md
```

### Manual — npm global

```bash
npm install -g oh-my-opencode
```

### Manual — npx (no install)

```bash
npx oh-my-opencode@latest
```

### Manual — bun

```bash
bun add -g oh-my-opencode
```

---

## Quick Start

```bash
# Launch the TUI
omo

# Or use the legacy alias
oh-my-opencode

# Once inside the TUI, activate full orchestration:
ultrawork
# or the short alias:
ulw
```

---

## Configuration

omo uses a `config.json` (OpenCode-compatible) plus a `CLAUDE.md` / `AGENTS.md` skill file in your project root.

### Minimal `~/.config/opencode/config.json`

```jsonc
{
  "$schema": "https://opencode.ai/config.schema.json",
  "theme": "opencode",
  "providers": {
    "anthropic": {
      "apiKey": "$ANTHROPIC_API_KEY"
    },
    "openai": {
      "apiKey": "$OPENAI_API_KEY"
    },
    "google": {
      "apiKey": "$GOOGLE_API_KEY"
    }
  },
  "model": "anthropic/claude-opus-4-5",
  "autoshare": false
}
```

### Multi-provider `config.json` (full power)

```jsonc
{
  "$schema": "https://opencode.ai/config.schema.json",
  "theme": "opencode",
  "providers": {
    "anthropic": {
      "apiKey": "$ANTHROPIC_API_KEY"
    },
    "openai": {
      "apiKey": "$OPENAI_API_KEY"
    },
    "google": {
      "apiKey": "$GOOGLE_API_KEY"
    },
    "moonshot": {
      "apiKey": "$MOONSHOT_API_KEY",
      "baseURL": "https://api.moonshot.ai/v1"
    },
    "zhipu": {
      "apiKey": "$ZHIPU_API_KEY",
      "baseURL": "https://open.bigmodel.cn/api/paas/v4"
    }
  },
  "model": "anthropic/claude-opus-4-5",
  "mcpServers": {
    "exa": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "exa-mcp-server"],
      "env": {
        "EXA_API_KEY": "$EXA_API_KEY"
      }
    },
    "context7": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

### Environment variables

```bash
# Required for your chosen provider(s)
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
export GOOGLE_API_KEY="..."

# Optional — for built-in MCPs
export EXA_API_KEY="..."

# Optional — additional providers
export MOONSHOT_API_KEY="..."   # Kimi
export ZHIPU_API_KEY="..."      # GLM / Z.ai
```

---

## Core Commands Inside the TUI

| Command | Alias | Description |
|---|---|---|
| `ultrawork` | `ulw` | Activates all discipline agents; orchestrates until task is complete |
| `/ulw-loop` | ralph loop | Self-referential loop; reruns until 100% done |
| `/new` | | Start a new session |
| `/model` | | Switch the active model |
| `/mcp` | | List or toggle MCP servers |
| `/share` | | Share session transcript |
| `Ctrl+C` | | Interrupt current agent run |

---

## Discipline Agents

`ultrawork` spins up the full Sisyphus team in parallel:

| Agent | Role |
|---|---|
| **Sisyphus** | Orchestrator — delegates and tracks overall task completion |
| **Hephaestus** | Implementer — writes and edits code |
| **Oracle** | Reasoner — deep analysis, planning, and debugging |
| **Librarian** | Documentation and context retrieval |
| **Explore** | Web search, discovery, and research via Exa MCP |

Each agent is scoped to a lightweight context to avoid token bloat.

---

## Skills / AGENTS.md

omo is Claude Code-compatible. Place a `CLAUDE.md` or `AGENTS.md` file in your project root to give agents project-specific context:

```markdown
# Project Context

## Stack
- Next.js 15 (App Router)
- Prisma + PostgreSQL
- Tailwind CSS v4

## Key commands
- `pnpm dev` — start dev server
- `pnpm test` — run Vitest suite
- `pnpm db:migrate` — run Prisma migrations

## Conventions
- All React components use named exports
- API routes live in `src/app/api/`
- Shared types in `src/types/`
```

---

## Hash-Anchored Edit Tool

omo uses `LINE#ID` content hashes to validate every file edit, preventing stale-line errors.

When writing agent prompts that involve edits, reference the hash pattern:

```typescript
// The agent will internally resolve LINE#ID before patching.
// You never interact with hashes directly — they are managed by omo's edit tool.
// Just describe the change in plain language:

"In src/utils/format.ts, change the date format from MM/DD/YYYY to YYYY-MM-DD"
```

---

## LSP + AST-Grep Integration

omo ships workspace-level rename and AST-aware rewrite capabilities:

```bash
# Agents can invoke these automatically. You can also trigger explicitly:
# "Rename all usages of `getUserById` to `fetchUserById` across the workspace"
# "Find all React components that use useEffect without a dependency array"
```

For manual AST-grep queries in your terminal:

```bash
# Install ast-grep if not present
npm install -g @ast-grep/cli

# Example: find all `console.log` calls in TypeScript files
ast-grep --lang typescript --pattern 'console.log($$$)'

# Example: find React hooks missing dependency arrays
ast-grep --lang typescript --pattern 'useEffect(() => { $$$ })'
```

---

## Built-in MCP Servers

| MCP | Purpose | Env var needed |
|---|---|---|
| **Exa** | Live web search | `EXA_API_KEY` |
| **Context7** | Official library docs lookup | none |
| **Grep.app** | GitHub code search | none |

Add them to `config.json` under `mcpServers`:

```jsonc
{
  "mcpServers": {
    "exa": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "exa-mcp-server"],
      "env": { "EXA_API_KEY": "$EXA_API_KEY" }
    },
    "context7": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    },
    "grep-app": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "grep-app-mcp"]
    }
  }
}
```

---

## Skill-Embedded MCPs

You can attach MCP server configuration directly inside a skill file so the agent carries its own tools without polluting the global context:

```markdown
<!-- Inside a custom SKILL.md -->
---
mcpServers:
  my-db-tool:
    type: stdio
    command: node
    args: ["./tools/db-inspector.js"]
    env:
      DATABASE_URL: "$DATABASE_URL"
---
```

---

## Tmux Integration

omo supports full interactive terminals via tmux, enabling REPLs, debuggers, and TUI apps inside the agent session.

```bash
# Ensure tmux is installed
brew install tmux      # macOS
sudo apt install tmux  # Debian/Ubuntu

# omo will detect tmux automatically when launched inside a tmux session
tmux new -s omo
omo
```

Agents can then spawn panes, run long-lived processes, and read their output.

---

## IntentGate

Before classifying or acting on any user message, omo's IntentGate layer analyses true intent. This prevents literal misinterpretation.

Practical implication — be expressive with your prompts:

```
# Less effective (too literal):
"Delete all TODO comments"

# More effective (expresses true intent):
"Clean up all TODO comments that are already resolved or no longer relevant,
but keep any that describe genuine future work"
```

---

## Common Patterns

### Pattern 1: Full task delegation

```
ultrawork

I need to add Stripe subscription billing to this Next.js app.
The app already has Prisma + PostgreSQL. Add:
- Checkout session creation
- Webhook handler for subscription events
- A /billing page showing current plan and invoices
```

### Pattern 2: Looping until complete

```
/ulw-loop

Refactor all class-based React components in src/components/ to
functional components with hooks. Do not stop until every file is done.
```

### Pattern 3: Multi-model reasoning task

```
# Switch to a reasoning-optimised model for planning
/model openai/o3

Analyse the current architecture in src/ and produce a migration plan
to move from REST to tRPC. List every file that needs changes.
```

### Pattern 4: Web-assisted implementation

```
ultrawork

Use Exa to find the latest Prisma v6 migration docs, then update
our schema.prisma to use the new multi-schema feature.
```

### Pattern 5: Background specialist

```
# Spawn a background linting agent while you work on features
ultrawork

In the background: run ESLint on the entire codebase, fix all
auto-fixable issues, and report the remaining ones. Don't block
my main session.
```

---

## Troubleshooting

### `omo` command not found after `npm install -g`

```bash
# Check npm global bin is on PATH
npm bin -g           # shows the bin dir
export PATH="$(npm bin -g):$PATH"  # add to ~/.bashrc or ~/.zshrc
```

### API key not recognised

```bash
# Verify the env var is exported (not just set)
echo $ANTHROPIC_API_KEY

# Re-export if empty
export ANTHROPIC_API_KEY="sk-ant-..."
```

### MCP server fails to start

```bash
# Test the MCP server outside omo
npx -y exa-mcp-server

# Check for missing env vars — omo passes env vars from config.json
# but they must resolve at launch time
echo $EXA_API_KEY
```

### Agent gets stuck / stops mid-task

Use the ralph loop to force continuation:

```
/ulw-loop

Continue the previous task from where it stopped. Do not stop
until fully complete.
```

### Hash-anchor edit errors

If you see stale-line or hash mismatch errors, the file was modified externally after the agent read it. Fix:

```
# Tell the agent explicitly:
"Re-read the file before making any edits"
```

### Tmux pane not interactive

```bash
# Ensure you launched omo from within a tmux session
tmux new -s work
omo
```

---

## Model Recommendations (cost/capability)

| Use case | Recommended model |
|---|---|
| Orchestration | `anthropic/claude-opus-4-5` or `moonshot/kimi-k2` |
| Deep reasoning / planning | `openai/o3` |
| Fast iteration | `google/gemini-2.5-flash` |
| Large codebase context | `google/gemini-2.5-pro` |
| Budget-conscious coding | `zhipu/glm-4` |

Switch mid-session with `/model <provider>/<model-id>`.

---

## Further Reading

- [Installation Guide](https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/refs/heads/dev/docs/guide/installation.md)
- [The Harness Problem — blog.can.ac](https://blog.can.ac/2026/02/12/the-harness-problem/)
- [Sisyphus Labs waitlist](https://sisyphuslabs.ai)
- [Discord community](https://discord.gg/PUwSMR9XNk)
- [DeepWiki docs](https://deepwiki.com/code-yeongyu/oh-my-openagent)
```
