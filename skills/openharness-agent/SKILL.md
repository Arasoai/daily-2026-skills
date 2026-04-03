```markdown
---
name: openharness-agent
description: Skill for using OpenHarness (oh) — the lightweight open-source Python agent harness with tool-use, skills, memory, and multi-agent coordination
triggers:
  - set up OpenHarness agent
  - use oh command for AI agent
  - configure openharness tools
  - build agent with openharness
  - add skills to openharness
  - create multi-agent workflow with openharness
  - troubleshoot openharness tool execution
  - extend openharness with custom plugins
---

# OpenHarness Agent Harness (`oh`)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

OpenHarness is a lightweight Python agent harness that wraps an LLM with tools, memory, permissions, and multi-agent coordination — 44x lighter than Claude Code at ~11,700 lines of Python. It exposes a single `oh` CLI and a composable Python API.

---

## Installation

**Requirements:** Python 3.10+, [uv](https://docs.astral.sh/uv/), Node.js 18+ (optional, for React TUI)

```bash
git clone https://github.com/HKUDS/OpenHarness.git
cd OpenHarness
uv sync --extra dev
```

Activate the environment or use `uv run` prefix for all commands.

---

## Environment Configuration

```bash
# Anthropic (default)
export ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY

# Moonshot / Kimi (Anthropic-compatible)
export ANTHROPIC_BASE_URL=https://api.moonshot.cn/anthropic
export ANTHROPIC_API_KEY=$MOONSHOT_API_KEY
export ANTHROPIC_MODEL=kimi-k2.5

# Vertex-compatible gateway
export ANTHROPIC_BASE_URL=https://your-vertex-gateway/anthropic
export ANTHROPIC_API_KEY=$GCP_API_KEY

# Bedrock-compatible gateway
export ANTHROPIC_BASE_URL=https://your-bedrock-gateway
export ANTHROPIC_API_KEY=$AWS_API_KEY
```

Provider detection is automatic based on `ANTHROPIC_BASE_URL` content (`moonshot`, `vertex`, `aiplatform`, `bedrock`).

---

## CLI Reference

### Interactive Mode

```bash
oh                   # launch interactive TUI (React+Ink terminal UI)
uv run oh            # without activating venv
```

### Non-Interactive / Scripted Mode

```bash
# Single prompt, plain text output
oh -p "Explain this codebase"

# JSON output (structured, for programmatic consumption)
oh -p "List all functions in main.py" --output-format json

# Streaming JSON events (real-time, line-delimited)
oh -p "Fix the bug in src/api.py" --output-format stream-json

# Pass a file as context
oh -p "Review this file" --file src/main.py

# Resume a previous session
oh --resume <session-id>

# Non-interactive with permission bypass (CI/automation)
oh -p "Run tests and fix failures" --permission-mode auto
```

### Key Slash Commands (inside interactive session)

```
/help              — list all 54 commands
/plan              — show or set current task plan
/commit            — stage and commit changes via git
/resume            — resume a previous session
/memory            — view or edit persistent MEMORY.md
/compact           — manually trigger context compression
/tools             — list available tools
/skills            — list loaded skills
/hooks             — show registered hooks
/agents            — list spawned subagents
/mcp               — manage MCP server connections
/config            — view or edit current configuration
/clear             — clear conversation context
/exit              — quit the session
```

---

## Architecture Overview

```
openharness/
  engine/        # Agent loop: query → stream → tool-call → loop
  tools/         # 43 built-in tools (file, shell, search, web, MCP)
  skills/        # On-demand .md skill loading
  plugins/       # Commands, hooks, agents, MCP servers
  permissions/   # Multi-level safety: path rules, command deny
  hooks/         # PreToolUse / PostToolUse lifecycle events
  commands/      # 54 slash commands
  mcp/           # Model Context Protocol client
  memory/        # Persistent cross-session MEMORY.md
  tasks/         # Background task lifecycle
  coordinator/   # Subagent spawning + team coordination
  prompts/       # System prompt assembly, CLAUDE.md injection
  config/        # Multi-layer config with migrations
  ui/            # React+Ink TUI backend protocol
```

---

## The Agent Loop (Python API)

```python
from openharness.engine import AgentLoop
from openharness.config import HarnessConfig

config = HarnessConfig.load()  # reads .openharness.json + env vars

loop = AgentLoop(config=config)

# Run a task programmatically
result = await loop.run(
    prompt="Refactor src/utils.py to use dataclasses",
    permission_mode="auto",        # bypass interactive approvals
    output_format="stream-json",   # text | json | stream-json
)

print(result.output)
print(f"Tokens used: {result.token_count}, Cost: ${result.cost:.4f}")
```

### Streaming Output

```python
import asyncio
from openharness.engine import AgentLoop
from openharness.config import HarnessConfig

async def stream_agent():
    loop = AgentLoop(config=HarnessConfig.load())

    async for event in loop.stream(prompt="Analyse main.py"):
        if event["type"] == "text":
            print(event["content"], end="", flush=True)
        elif event["type"] == "tool_use":
            print(f"\n[Tool] {event['name']}({event['input']})")
        elif event["type"] == "tool_result":
            print(f"[Result] {event['content'][:120]}")
        elif event["type"] == "done":
            print(f"\nDone. Cost: ${event['cost']:.4f}")

asyncio.run(stream_agent())
```

---

## Tools (43 Built-in)

Categories and representative tools:

| Category | Tools |
|----------|-------|
| **File I/O** | `read_file`, `write_file`, `edit_file`, `list_directory`, `find_files` |
| **Shell** | `bash`, `run_command`, `run_script` |
| **Search** | `grep`, `semantic_search`, `glob` |
| **Web** | `web_fetch`, `web_search` |
| **MCP** | Any MCP server tool via `mcp_call` |
| **Agent** | `spawn_agent`, `delegate_task` |

### Calling a Tool Directly (Python)

```python
from openharness.tools import ToolRegistry

registry = ToolRegistry()

# Read a file
result = await registry.execute("read_file", {"path": "src/main.py"})
print(result.content)

# Run a shell command
result = await registry.execute("bash", {"command": "pytest tests/ -q"})
print(result.content)

# Fetch a URL
result = await registry.execute("web_fetch", {"url": "https://example.com"})
print(result.content[:500])
```

---

## Skills (On-Demand Knowledge Loading)

Skills are `.md` files injected into the system prompt on demand. Compatible with `anthropics/skills` and `claude-code/plugins` formats.

### Installing a Skill

```bash
# Place skill file in the skills directory
cp my-skill.md ~/.openharness/skills/my-skill.md

# Or project-local
cp my-skill.md .openharness/skills/my-skill.md
```

### Writing a Custom Skill

```markdown
---
name: my-project-patterns
description: Coding patterns for MyProject
triggers:
  - use myproject conventions
  - myproject coding style
---

# MyProject Patterns

Always use `MyProject.create()` factory, never direct constructors.
Prefer async context managers for resource handling.
...
```

### Loading Skills Programmatically

```python
from openharness.skills import SkillLoader

loader = SkillLoader(skill_dirs=["~/.openharness/skills", ".openharness/skills"])

# List available skills
skills = loader.list_skills()
for skill in skills:
    print(f"{skill.name}: {skill.description}")

# Load specific skill into context
content = loader.load("my-project-patterns")
print(content)
```

---

## Memory (Persistent Cross-Session)

OpenHarness reads and writes `MEMORY.md` (and `CLAUDE.md`) for persistent context.

```bash
# View current memory
/memory

# The agent auto-updates MEMORY.md with important facts
# You can also edit it directly
vim ~/.openharness/MEMORY.md
```

```python
from openharness.memory import MemoryStore

memory = MemoryStore()

# Read persisted memory
content = memory.read()
print(content)

# Write a fact to persist across sessions
memory.append("## Project Notes\n- Always run `make lint` before committing.")
```

**CLAUDE.md** is auto-discovered from the project root and injected as system context:

```bash
# Project-level context injection
echo "# Project Rules\nUse Black for formatting. Python 3.11+." > CLAUDE.md
```

---

## Permissions & Governance

### Permission Modes

| Mode | Behaviour |
|------|-----------|
| `ask` | Prompt user for every tool call (default interactive) |
| `auto` | Execute all tools without prompting (CI/automation) |
| `restricted` | Only allow read operations; block write/shell |

```bash
oh -p "Run all tests" --permission-mode auto
oh -p "Review code only" --permission-mode restricted
```

### Configuration File (`.openharness.json`)

```json
{
  "permission_mode": "ask",
  "allowed_paths": ["./src", "./tests"],
  "denied_commands": ["rm -rf", "curl | bash"],
  "denied_paths": ["/etc", "/usr"],
  "hooks": {
    "pre_tool_use": ["./hooks/log_tool.py"],
    "post_tool_use": ["./hooks/audit.py"]
  }
}
```

---

## Hooks (PreToolUse / PostToolUse)

```python
# hooks/log_tool.py
import sys, json

event = json.load(sys.stdin)   # {"tool": "bash", "input": {"command": "..."}}

tool_name = event["tool"]
if tool_name == "bash" and "rm" in event["input"].get("command", ""):
    # Block the tool call by exiting non-zero
    print(json.dumps({"block": True, "reason": "rm not allowed"}))
    sys.exit(1)

# Allow by default
print(json.dumps({"block": False}))
sys.exit(0)
```

Register hooks in `.openharness.json` or programmatically:

```python
from openharness.hooks import HookRegistry

registry = HookRegistry()

@registry.pre_tool_use
async def my_hook(event):
    print(f"About to call: {event.tool_name}")
    if event.tool_name == "bash":
        cmd = event.input.get("command", "")
        if "drop table" in cmd.lower():
            return {"block": True, "reason": "DB drop blocked"}
    return {"block": False}
```

---

## Multi-Agent Coordination

### Spawning a Subagent

```python
from openharness.coordinator import AgentCoordinator

coordinator = AgentCoordinator()

# Spawn and await a subagent
result = await coordinator.spawn(
    prompt="Write unit tests for src/parser.py",
    tools=["read_file", "write_file", "bash"],
    permission_mode="auto",
)
print(result.output)
```

### Background Tasks

```python
from openharness.tasks import TaskManager

tasks = TaskManager()

# Start a background task
task_id = await tasks.start(
    name="run-ci",
    prompt="Run the full test suite and report failures",
    permission_mode="auto",
)

# Poll status
status = await tasks.status(task_id)
print(status)  # pending | running | done | failed

# Await completion
result = await tasks.wait(task_id)
print(result.output)
```

---

## MCP (Model Context Protocol) Integration

```json
// .openharness.json — register an MCP server
{
  "mcp_servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"}
    }
  }
}
```

```python
from openharness.mcp import MCPClient

client = MCPClient(server_name="filesystem")
await client.connect()

tools = await client.list_tools()
result = await client.call_tool("read_file", {"path": "/workspace/README.md"})
print(result)
```

---

## Context Compression (Auto-Compact)

When the context window fills, OpenHarness auto-compacts the conversation:

```bash
# Trigger manually inside session
/compact

# Configure threshold in .openharness.json
```

```json
{
  "context": {
    "auto_compact": true,
    "compact_threshold": 0.85
  }
}
```

---

## Common Patterns

### CI/CD Automation

```bash
#!/bin/bash
# .github/scripts/agent-review.sh
oh -p "Review the diff in this PR and list any bugs" \
   --output-format json \
   --permission-mode restricted \
   | jq '.findings'
```

### Project Bootstrap

```bash
oh -p "Inspect this repository and create a CLAUDE.md with project conventions"
```

### Iterative Fix Loop

```python
import asyncio
from openharness.engine import AgentLoop
from openharness.config import HarnessConfig

async def fix_until_green():
    loop = AgentLoop(config=HarnessConfig.load())
    result = await loop.run(
        prompt="Run pytest. If any tests fail, fix the code and re-run until all pass.",
        permission_mode="auto",
    )
    print(result.output)

asyncio.run(fix_until_green())
```

---

## Troubleshooting

### `oh: command not found`

```bash
# Use uv run instead
uv run oh -p "hello"

# Or activate the venv first
source .venv/bin/activate
oh -p "hello"
```

### API authentication errors

```bash
# Confirm env vars are exported (not just set)
export ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY
echo $ANTHROPIC_API_KEY   # must be non-empty

# For custom providers, verify base URL format
export ANTHROPIC_BASE_URL=https://api.moonshot.cn/anthropic  # no trailing slash
```

### Tool execution blocked unexpectedly

```bash
# Check permission mode
oh --permission-mode auto -p "your prompt"

# Check denied_commands and denied_paths in .openharness.json
cat .openharness.json | jq '.denied_commands, .denied_paths'
```

### Context window full / slow responses

```bash
# Manually compact inside session
/compact

# Lower auto-compact threshold in config
# "compact_threshold": 0.75
```

### Subagent not returning

```python
# Add a timeout
result = await coordinator.spawn(
    prompt="...",
    timeout=120,   # seconds
)
```

### Running Tests

```bash
# Unit tests (114 tests)
uv run pytest tests/unit/ -q

# E2E suites (6 suites)
uv run pytest tests/e2e/ -q

# Specific subsystem
uv run pytest tests/unit/test_tools.py -v
```

---

## Key Files to Know

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Project-level context auto-injected into every session |
| `~/.openharness/MEMORY.md` | Persistent cross-session memory |
| `.openharness.json` | Project config (permissions, hooks, MCP servers) |
| `~/.openharness/config.json` | Global user config |
| `src/openharness/api/provider.py` | Provider detection logic |
| `src/openharness/engine/loop.py` | Core agent loop implementation |
| `src/openharness/tools/` | All 43 tool implementations |
```
