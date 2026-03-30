```markdown
---
name: hitcc-claude-code-reverse-engineering
description: Documentation knowledge base for reverse-engineering Claude Code CLI v2.1.84 — covers runtime logic, agent loop, tool use, MCP, plugin/skill systems, and rewrite architecture
triggers:
  - how does claude code cli work internally
  - reverse engineer claude code
  - understand claude code agent loop
  - claude code tool use implementation
  - claude code mcp integration details
  - rewrite claude code cli
  - claude code session persistence logic
  - claude code prompt assembly pipeline
---

# HitCC — Claude Code CLI Reverse-Engineering Knowledge Base

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

HitCC is a structured documentation knowledge base that reverse-engineers the full runtime logic of **Claude Code CLI v2.1.84** (Node.js). It is not source code — it is topic-oriented analysis covering startup, agent loop, tool execution, prompt assembly, session persistence, MCP, plugins, skills, TUI, and rewrite architecture.

Use this skill when you need to understand how Claude Code works internally, build a compatible alternative, or reference its architecture for your own agentic coding shell.

---

## What HitCC Covers

| Area | Topics |
|---|---|
| Runtime | CLI entry, command tree, mode dispatch, session/transcript persistence |
| Execution | Agent Loop, tool execution core, Hook runtime, Permission/Sandbox/Approval |
| Prompt | Input compilation, prompt assembly, context layering, attachment lifecycle |
| Model | Model adapter, provider selection, auth, stream handling, remote transport |
| Ecosystem | MCP, Plugin, Skill, TUI, Remote persistence, Bridge, Plan system |
| Rewrite | Candidate layering, directory skeleton, open questions, blocking judgments |
| Network | `web-search`, `web-fetch`, telemetry, control plane |
| Settings | Sources, paths, merging, caching, write-back, key consumption surfaces |

---

## Getting the Analyzed Package

HitCC documents Claude Code CLI v2.1.84. To obtain the exact package analyzed:

```bash
npm pack @anthropic-ai/claude-code@2.1.84
```

This downloads `anthropic-ai-claude-code-2.1.84.tgz` for static analysis. HitCC does **not** redistribute original source.

---

## Installation / Setup

HitCC is a documentation repository — no package install required.

```bash
git clone https://github.com/hitmux/HitCC.git
cd HitCC
```

### Optional: Run Recovery Tools

Python scripts under `recovery_tools/` perform initial cleanup on obfuscated/encrypted source:

```python
# recovery_tools/ scripts — Python 3.x required
# Example: run a cleanup pass on unpacked CLI source
python recovery_tools/cleanup.py --input ./unpacked_cli --output ./cleaned_cli
```

> No external dependencies are documented; use a standard Python 3 environment.

---

## Recommended Reading Order

```
docs/
├── 00-overview/
│   ├── 00-index.md                          ← Start here for global entry
│   ├── 01-scope-and-evidence.md             ← What is known vs. unknown
│   └── 02-document-style-and-structure-conventions.md
├── 01-runtime/
│   ├── 01-product-cli-and-modes.md          ← CLI shape, command tree, mode dispatch
│   ├── 02-session-transcript-persistence.md
│   ├── 03-input-compilation-and-agent-loop.md
│   ├── 04-model-adapter-and-provider.md
│   ├── 05-web-search-and-web-fetch.md
│   ├── 06-telemetry-and-control-plane.md
│   └── 07-settings-system.md
├── 02-execution/
│   ├── 01-tool-execution-core.md
│   ├── 02-hook-runtime-and-permissions.md
│   ├── 03-prompt-assembly-and-context.md
│   └── 04-attachments-and-tool-use-context.md
├── 03-ecosystem/
│   ├── 01-resume-fork-sidechain-subagent.md
│   ├── 02-remote-persistence-and-bridge.md
│   ├── 03-mcp-integration.md
│   ├── 04-skill-and-plugin.md
│   └── 05-tui-runtime.md
├── 04-rewrite/
│   ├── 01-candidate-architecture.md
│   └── 02-open-questions-and-judgment.md
└── 05-appendix/
    ├── glossary.md
    └── evidence-map.md
```

**Fastest path:**
1. `docs/00-overview/01-scope-and-evidence.md` — confidence boundaries
2. `docs/01-runtime/01-product-cli-and-modes.md` — overall product shape
3. `docs/02-execution/` — agent loop, tools, prompts
4. `docs/03-ecosystem/` — MCP, plugin, skill, TUI
5. `docs/04-rewrite/` — engineering strategy

---

## Key Architectural Concepts

### Agent Loop (from docs/01-runtime/03-input-compilation-and-agent-loop.md)

Claude Code's core loop follows this pattern:

```
User Input
  → Input Compilation Pipeline
    → Context assembly (system prompt + rules + attachments)
    → Tool definitions injection
  → LLM Request (streaming)
    → Stream handler collects tool_use blocks
  → Tool Execution Core
    → Concurrent execution with permission checks
    → Hook runtime (pre/post hooks)
  → Result injection → next loop iteration
  → Compact branch (context window management)
```

### Tool Execution Core

Tools run with a concurrent execution model gated by the Permission/Sandbox/Approval system:

```python
# Conceptual rewrite pattern based on HitCC docs/02-execution/01-tool-execution-core.md

import asyncio
from typing import Any

async def execute_tools_concurrent(
    tool_calls: list[dict],
    permission_checker,
    hook_runner,
) -> list[dict]:
    """
    Mirrors Claude Code's concurrent tool dispatch with permission gating.
    """
    async def run_single(tool_call: dict) -> dict:
        tool_name = tool_call["name"]
        tool_input = tool_call["input"]

        # Pre-execution hook
        await hook_runner.run_pre_hook(tool_name, tool_input)

        # Permission check (may trigger approval UI)
        allowed = await permission_checker.check(tool_name, tool_input)
        if not allowed:
            return {"tool_use_id": tool_call["id"], "error": "permission_denied"}

        # Dispatch to tool implementation
        result = await dispatch_tool(tool_name, tool_input)

        # Post-execution hook
        await hook_runner.run_post_hook(tool_name, result)

        return {"tool_use_id": tool_call["id"], "content": result}

    return await asyncio.gather(*[run_single(tc) for tc in tool_calls])
```

### Session / Transcript Persistence

```python
# Pattern from docs/01-runtime/02-session-transcript-persistence.md

import json
import os
from pathlib import Path
from datetime import datetime

TRANSCRIPT_DIR = Path.home() / ".claude" / "transcripts"

def persist_turn(session_id: str, turn: dict) -> None:
    """Append a conversation turn to the session transcript."""
    session_file = TRANSCRIPT_DIR / f"{session_id}.jsonl"
    session_file.parent.mkdir(parents=True, exist_ok=True)
    with open(session_file, "a") as f:
        f.write(json.dumps(turn) + "\n")

def load_transcript(session_id: str) -> list[dict]:
    """Load all turns for session recovery/resume."""
    session_file = TRANSCRIPT_DIR / f"{session_id}.jsonl"
    if not session_file.exists():
        return []
    with open(session_file) as f:
        return [json.loads(line) for line in f if line.strip()]

def fork_session(source_id: str, new_id: str, fork_at_turn: int) -> None:
    """Fork a session at a specific turn index."""
    turns = load_transcript(source_id)
    forked = turns[:fork_at_turn]
    for turn in forked:
        persist_turn(new_id, turn)
```

### Settings System

```python
# Pattern from docs/01-runtime/07-settings-system.md

import json
import os
from pathlib import Path
from typing import Any

SETTINGS_PATHS = [
    Path("/etc/claude/settings.json"),               # system-wide
    Path.home() / ".claude" / "settings.json",       # user-level
    Path(".claude") / "settings.json",                # project-level (cwd)
]

def load_merged_settings() -> dict:
    """
    Load and merge settings from all sources.
    Later sources override earlier ones (project > user > system).
    """
    merged: dict[str, Any] = {}
    for path in SETTINGS_PATHS:
        if path.exists():
            with open(path) as f:
                layer = json.load(f)
            merged = deep_merge(merged, layer)
    # Environment variable overrides
    if api_key := os.environ.get("ANTHROPIC_API_KEY"):
        merged.setdefault("api", {})["key"] = api_key
    return merged

def deep_merge(base: dict, override: dict) -> dict:
    result = dict(base)
    for k, v in override.items():
        if isinstance(v, dict) and isinstance(result.get(k), dict):
            result[k] = deep_merge(result[k], v)
        else:
            result[k] = v
    return result
```

### Prompt Assembly and Context Layering

```python
# Pattern from docs/02-execution/03-prompt-assembly-and-context.md

from dataclasses import dataclass, field
from typing import Optional

@dataclass
class PromptContext:
    system_base: str = ""
    rules: list[str] = field(default_factory=list)        # CLAUDE.md rules
    skill_instructions: list[str] = field(default_factory=list)
    tool_definitions: list[dict] = field(default_factory=list)
    attachments: list[dict] = field(default_factory=list)  # files, images
    conversation_history: list[dict] = field(default_factory=list)

def assemble_system_prompt(ctx: PromptContext) -> str:
    """
    Mirrors Claude Code's layered system prompt assembly.
    Order: base → rules → skill instructions → attachment summaries
    """
    parts = [ctx.system_base]
    if ctx.rules:
        parts.append("\n\n## Project Rules\n" + "\n".join(f"- {r}" for r in ctx.rules))
    if ctx.skill_instructions:
        parts.append("\n\n## Skills\n" + "\n\n".join(ctx.skill_instructions))
    return "\n".join(filter(None, parts))

def discover_rules(cwd: str) -> list[str]:
    """
    Walk directory tree upward collecting CLAUDE.md rule files.
    Mirrors docs/02-execution/03-prompt-assembly-and-context.md instruction discovery.
    """
    rules = []
    path = Path(cwd)
    for parent in [path, *path.parents]:
        candidate = parent / "CLAUDE.md"
        if candidate.exists():
            rules.append(candidate.read_text())
    return list(reversed(rules))  # root rules first
```

### MCP Integration

```python
# Pattern from docs/03-ecosystem/03-mcp-integration.md

import httpx
from typing import Any

class MCPClient:
    """
    Minimal MCP (Model Context Protocol) client mirroring Claude Code's integration.
    """
    def __init__(self, server_url: str, api_key: str | None = None):
        self.server_url = server_url.rstrip("/")
        self.headers = {"Authorization": f"Bearer {api_key}"} if api_key else {}

    async def list_tools(self) -> list[dict]:
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{self.server_url}/tools", headers=self.headers)
            resp.raise_for_status()
            return resp.json()["tools"]

    async def call_tool(self, tool_name: str, arguments: dict) -> Any:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{self.server_url}/tools/{tool_name}",
                json={"arguments": arguments},
                headers=self.headers,
            )
            resp.raise_for_status()
            return resp.json()["result"]

# MCP server config pattern (mirrors .claude/settings.json mcp section)
MCP_CONFIG_EXAMPLE = {
    "mcpServers": {
        "my-server": {
            "url": "http://localhost:3100",
            "transport": "http",
        }
    }
}
```

### Compact Branch (Context Window Management)

```python
# Pattern from docs/01-runtime/03-input-compilation-and-agent-loop.md (compact branch)

def should_compact(history: list[dict], token_estimate: int, threshold: int = 180_000) -> bool:
    """
    Determines if the agent loop should trigger a compact (summarization) pass.
    Claude Code uses a token threshold on the running context.
    """
    return token_estimate >= threshold

async def compact_history(
    history: list[dict],
    summarize_fn,  # async fn(messages) -> str
) -> list[dict]:
    """
    Compact conversation history by summarizing older turns.
    Preserves the most recent N turns verbatim.
    """
    PRESERVE_RECENT = 10
    if len(history) <= PRESERVE_RECENT:
        return history

    older = history[:-PRESERVE_RECENT]
    recent = history[-PRESERVE_RECENT:]

    summary = await summarize_fn(older)
    summary_message = {
        "role": "user",
        "content": f"[Conversation summary]\n{summary}"
    }
    return [summary_message] + recent
```

---

## Configuration Reference

### Project-level `.claude/settings.json`

```json
{
  "model": "claude-opus-4-5",
  "maxTokens": 8192,
  "permissions": {
    "allowedTools": ["read_file", "write_file", "bash"],
    "requireApproval": ["bash", "write_file"]
  },
  "hooks": {
    "preToolUse": "./hooks/pre_tool.sh",
    "postToolUse": "./hooks/post_tool.sh"
  },
  "mcpServers": {
    "my-mcp": {
      "url": "http://localhost:3100",
      "transport": "http"
    }
  },
  "skills": ["./skills/my-skill.md"],
  "telemetry": {
    "enabled": false
  }
}
```

### Environment Variables

```bash
# Authentication
export ANTHROPIC_API_KEY="..."          # Primary API key

# Proxy / transport overrides
export ANTHROPIC_BASE_URL="..."         # Custom API base URL

# Telemetry opt-out
export CLAUDE_TELEMETRY_DISABLED=1

# Session directory override
export CLAUDE_TRANSCRIPT_DIR="/tmp/claude_sessions"
```

---

## Rewrite Architecture Skeleton

From `docs/04-rewrite/01-candidate-architecture.md`:

```
my_claude_rewrite/
├── cli/
│   ├── entry.py              # argparse command tree, mode dispatch
│   └── modes.py              # interactive / headless / pipe modes
├── runtime/
│   ├── agent_loop.py         # main loop, compact branch
│   ├── session.py            # transcript persistence, resume, fork
│   └── settings.py           # layered settings loader
├── execution/
│   ├── tool_core.py          # concurrent executor, permission gate
│   ├── hooks.py              # pre/post hook runner
│   ├── permissions.py        # sandbox, approval UI
│   └── prompt_assembly.py    # system prompt builder, rule discovery
├── model/
│   ├── adapter.py            # provider abstraction
│   ├── stream.py             # SSE stream handler
│   └── auth.py               # key resolution, refresh
├── tools/
│   ├── read_file.py
│   ├── write_file.py
│   ├── bash.py
│   ├── web_search.py
│   └── web_fetch.py
├── ecosystem/
│   ├── mcp_client.py
│   ├── plugin_loader.py
│   ├── skill_loader.py
│   └── tui.py
└── docs/                     # link or copy of HitCC docs/
```

---

## Common Patterns and Troubleshooting

### Pattern: Subagent / Sidechain Dispatch

```python
# From docs/03-ecosystem/01-resume-fork-sidechain-subagent.md

async def spawn_subagent(
    task: str,
    context: PromptContext,
    parent_session_id: str,
    agent_runner,
) -> str:
    """
    Spawn a child agent for a delegated subtask.
    Child session is forked from parent context.
    """
    child_session_id = f"{parent_session_id}__sub_{id(task)}"
    child_context = PromptContext(
        system_base=context.system_base,
        rules=context.rules,
        tool_definitions=context.tool_definitions,
        conversation_history=[],  # fresh history for subagent
    )
    result = await agent_runner.run(
        session_id=child_session_id,
        initial_message=task,
        context=child_context,
    )
    return result
```

### Troubleshooting: Unknown Behaviors

HitCC documents confidence levels for each area. For behaviors not yet documented:

1. Check `docs/00-overview/01-scope-and-evidence.md` — is it in the "unknown" list?
2. Check `docs/04-rewrite/02-open-questions-and-judgment.md` — is it a blocking open question?
3. Check `docs/05-appendix/evidence-map.md` — what evidence exists?

### Troubleshooting: Recovery Tools Producing Garbled Output

```bash
# Ensure you're on Python 3.10+
python --version

# Run cleanup with explicit encoding
python recovery_tools/cleanup.py \
  --input ./unpacked_cli \
  --output ./cleaned_cli \
  --encoding utf-8
```

### Troubleshooting: MCP Server Not Connecting

- Verify `mcpServers` config in `.claude/settings.json`
- Check `transport` field: `"http"` or `"stdio"` are the documented transports
- MCP tools are injected into `tool_definitions` during prompt assembly — confirm the tool list endpoint returns valid JSON

---

## Current Limitations (from HitCC docs)

| Area | Status |
|---|---|
| Original file tree | Not reconstructed — docs are topic-organized |
| Private server-side logic | Black-box — cannot be fully reconstructed |
| Edge-case behavior parity | Not guaranteed |
| Runtime dynamic analysis | Not included — static analysis only |
| Direct runnable CLI | Not provided by HitCC |

For full confidence boundaries: `docs/00-overview/01-scope-and-evidence.md`  
For blocking engineering questions: `docs/04-rewrite/02-open-questions-and-judgment.md`

---

## License

HitCC documentation is licensed under **CC BY 4.0**.  
Claude Code CLI is a product of Anthropic PBC. HitCC has no affiliation with Anthropic PBC.
```
