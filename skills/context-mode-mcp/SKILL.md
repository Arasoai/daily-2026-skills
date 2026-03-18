```markdown
---
name: context-mode-mcp
description: Privacy-first MCP virtualization layer that sandboxes tool calls to save context window space and maintain session continuity across compaction
triggers:
  - "set up context mode"
  - "install context-mode MCP"
  - "reduce context window usage"
  - "sandbox MCP tool calls"
  - "session continuity after compaction"
  - "context-mode not routing tools"
  - "configure context mode hooks"
  - "optimize AI agent context"
---

# Context Mode MCP

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Context Mode is a privacy-first MCP server that acts as a virtualization layer between AI coding agents and their tools. It solves two problems: (1) raw tool output floods the context window (a Playwright snapshot = 56 KB, 20 GitHub issues = 59 KB), and (2) conversation compaction causes agents to forget session state. Context Mode sandboxes tool calls in subprocesses — raw data never enters context — and tracks all activity in local SQLite with FTS5 search for retrieval after compaction. Nothing leaves your machine; no telemetry, no cloud sync.

## Installation

### Claude Code (Full Plugin — Recommended)

```bash
/plugin marketplace add mksglu/context-mode
/plugin install context-mode@context-mode
```

Restart Claude Code. The plugin auto-installs:
- MCP server with 6 sandbox tools
- PreToolUse/PostToolUse/PreCompact/SessionStart hooks
- `CLAUDE.md` routing instructions in your project root
- Slash commands (`/context-mode:ctx-stats`, `/context-mode:ctx-doctor`, `/context-mode:ctx-upgrade`)

### Claude Code (MCP-only, no hooks)

```bash
claude mcp add context-mode -- npx -y context-mode
```

### Other Platforms (Global Install)

```bash
npm install -g context-mode
```

### Gemini CLI

Add to `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "context-mode": {
      "command": "context-mode"
    }
  },
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "",
        "hooks": [{ "type": "command", "command": "context-mode hook gemini-cli beforetool" }]
      }
    ],
    "AfterTool": [
      {
        "matcher": "",
        "hooks": [{ "type": "command", "command": "context-mode hook gemini-cli aftertool" }]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [{ "type": "command", "command": "context-mode hook gemini-cli sessionstart" }]
      }
    ]
  }
}
```

### VS Code Copilot

Create `.vscode/mcp.json`:

```json
{
  "servers": {
    "context-mode": {
      "command": "context-mode"
    }
  }
}
```

Create `.github/hooks/context-mode.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      { "type": "command", "command": "context-mode hook vscode-copilot pretooluse" }
    ],
    "PostToolUse": [
      { "type": "command", "command": "context-mode hook vscode-copilot posttooluse" }
    ],
    "SessionStart": [
      { "type": "command", "command": "context-mode hook vscode-copilot sessionstart" }
    ]
  }
}
```

### Cursor

Create `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "context-mode": {
      "command": "context-mode"
    }
  }
}
```

Create `.cursor/hooks.json`:

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "command": "context-mode hook cursor pretooluse",
        "matcher": "Shell|Read|Grep|WebFetch|Task|MCP:ctx_execute|MCP:ctx_execute_file|MCP:ctx_batch_execute"
      }
    ],
    "postToolUse": [
      {
        "command": "context-mode hook cursor posttooluse"
      }
    ]
  }
}
```

### OpenCode

Add to `opencode.json` in project root:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context-mode": {
      "type": "local",
      "command": ["context-mode"]
    }
  },
  "plugin": ["context-mode"]
}
```

## The 6 Sandbox MCP Tools

### `ctx_execute` — Run a shell command in sandbox

```javascript
// Instead of: Bash("gh issue list --limit 20")  → 59 KB in context
// Use:
ctx_execute({ command: "gh issue list --limit 20" })
// Returns: indexed summary + search handle, ~1.2 KB in context
```

### `ctx_execute_file` — Read and index a file in sandbox

```javascript
// Instead of: Read("./logs/access.log")  → 45 KB in context
// Use:
ctx_execute_file({ path: "./logs/access.log" })
// Returns: file summary + FTS5 index handle
```

### `ctx_batch_execute` — Run multiple commands, one context entry

```javascript
ctx_batch_execute({
  commands: [
    "git log --oneline -20",
    "git diff HEAD~1",
    "git status"
  ]
})
// All three results indexed together, single small context entry
```

### `ctx_fetch_and_index` — Fetch a URL and sandbox the response

```javascript
// Instead of: WebFetch("https://api.github.com/repos/owner/repo/issues")
// Use:
ctx_fetch_and_index({ url: "https://api.github.com/repos/owner/repo/issues" })
// Playwright snapshots, API responses, HTML — all sandboxed
```

### `ctx_index` — Manually index arbitrary content

```javascript
ctx_index({
  content: largeRawString,
  label: "deployment-logs-2026-03-18"
})
// Stores in SQLite FTS5, returns search handle
```

### `ctx_search` — Query indexed content via BM25

```javascript
ctx_search({
  query: "authentication error 401",
  limit: 5
})
// Returns top BM25-ranked results from current session's index
// This is how the agent recovers state after compaction
```

## Slash Commands (Claude Code Only)

| Command | Purpose |
|---|---|
| `/context-mode:ctx-stats` | Per-tool savings breakdown, tokens consumed, savings ratio |
| `/context-mode:ctx-doctor` | Diagnostics — runtimes, hooks, FTS5, plugin registration, versions |
| `/context-mode:ctx-upgrade` | Pull latest, rebuild, migrate cache, fix hooks |

On other platforms these work as MCP tools — just invoke `ctx stats`, `ctx doctor`, or `ctx upgrade` by name.

## Session Continuity Pattern

Context Mode tracks file edits, git operations, tasks, errors, and decisions in SQLite. After compaction:

1. The agent calls `ctx_search` with relevant keywords
2. BM25 ranking surfaces the most relevant prior events
3. The agent resumes without re-reading files or re-running commands

```javascript
// After compaction — recovering state
ctx_search({ query: "currently editing authentication middleware" })
// Returns: last known file path, recent edits, pending tasks

ctx_search({ query: "unresolved errors last session" })
// Returns: error events with file/line context
```

**Session lifecycle:**
- `--continue` flag: resumes previous session, keeps SQLite data
- No `--continue`: previous session data is deleted immediately (fresh slate)

## Routing Instructions Auto-Management

On SessionStart, context-mode writes routing instructions to your project:

| Platform | File |
|---|---|
| Claude Code | `CLAUDE.md` |
| Gemini CLI | `GEMINI.md` |
| VS Code Copilot | `.github/copilot-instructions.md` |
| Cursor / Codex | Project root instruction file |

**Behavior is idempotent:**
- File missing → written fresh
- File exists, no context-mode rules → rules appended, existing content preserved
- File has context-mode rules → skipped, no duplicates

## Why Hooks Matter

Without hooks, routing compliance is ~60% — the model sometimes ignores instructions and runs raw `curl`, reads large files directly, or dumps unprocessed output. One unrouted Playwright snapshot (56 KB) wipes an entire session's savings.

With hooks, every tool call is intercepted before execution:
- Dangerous commands (`curl`, `wget`, direct large file reads) are blocked
- Routing guidance is injected at the call site in real time
- Compliance reaches ~98% context savings

## Real Savings Example

```
Without context-mode:
  Playwright snapshot:     56 KB → context
  gh issue list (20):      59 KB → context
  access.log read:         45 KB → context
  Total after 30 min:     ~315 KB consumed (~40% of context)

With context-mode:
  All three sandboxed:     ~5.4 KB → context
  Savings:                 98% reduction
```

## Configuration Reference

Context-mode is zero-config for most use cases. The MCP server auto-starts via `npx -y context-mode` or the global `context-mode` binary.

SQLite databases are stored in `~/.context-mode/` and scoped per session. No environment variables required. No API keys. No accounts.

## Troubleshooting

### Tools not being routed to sandbox

1. Run `/context-mode:ctx-doctor` (Claude Code) or `ctx_doctor` (other platforms)
2. Verify hooks are registered: check platform-specific hook config files exist
3. Confirm `CLAUDE.md`/`GEMINI.md`/routing file exists in project root with context-mode rules
4. Without hooks, model compliance drops to ~60% — hooks are essential for reliable routing

### MCP server not found

```bash
# Verify global install
which context-mode
context-mode --version

# Or use npx fallback
npx -y context-mode
```

### ctx_search returns no results

- Ensure you're in a `--continue` session (fresh sessions start with empty index)
- Verify tools were called via sandbox (not raw Bash/Read)
- Run `ctx_stats` to confirm events were indexed in current session

### Hooks not firing on Cursor

- `sessionStart` hook is not supported by Cursor's validator — routing instructions are delivered via MCP server startup instead
- Ensure `.cursor/hooks.json` uses `preToolUse`/`postToolUse` only
- Project `.cursor/hooks.json` overrides `~/.cursor/hooks.json` — check for conflicts

### Plugin upgrade

```bash
# Claude Code
/context-mode:ctx-upgrade

# Other platforms
npm update -g context-mode
```

## Privacy Architecture

- All raw data processed in sandboxed subprocesses
- SQLite lives in `~/.context-mode/` — local only
- No network calls, no telemetry, no cloud sync
- No account, no API key, no per-seat subscription
- Sessions are ephemeral by default — data deleted unless `--continue` is passed
- License: Elastic License v2 (ELv2)
```
