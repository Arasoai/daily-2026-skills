```markdown
---
name: harness-engineering-cc-to-ai-coding
description: Expert skill for reading, navigating, and applying insights from the "Harness Engineering" book — a deep technical analysis of Claude Code's architecture, agent loops, prompt engineering, context management, and AI coding best practices.
triggers:
  - "help me understand claude code architecture"
  - "explain agent loop from harness engineering"
  - "how does claude code handle context management"
  - "what does the horse book say about prompt caching"
  - "harness engineering best practices"
  - "claude code permission system explained"
  - "how to build ai coding agents like claude code"
  - "马书 claude code 源码分析"
---

# Harness Engineering: From Claude Code to AI Coding

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

**Harness Engineering** (中文别名:《马书》) is a Chinese technical book that analyzes Claude Code `v2.1.88`'s publicly released package and source map reconstructions. It extracts architectural patterns, context strategies, permission systems, and production practices for building AI coding agents.

- **Online reading (Chinese):** https://zhanghandong.github.io/harness-engineering-from-cc-to-ai-coding/
- **Online reading (English preview):** https://zhanghandong.github.io/harness-engineering-from-cc-to-ai-coding/en/
- **GitHub:** https://github.com/ZhangHanDong/harness-engineering-from-cc-to-ai-coding

This is a **book project** built with [mdBook](https://rust-lang.github.io/mdBook/). The primary deliverable is the book itself, not a library or CLI tool you integrate into code.

---

## Local Setup & Preview

### Prerequisites

Install [mdBook](https://rust-lang.github.io/mdBook/):

```bash
# Via Rust/Cargo
cargo install mdbook

# Via Homebrew (macOS)
brew install mdbook

# Via pre-built binary (Linux/macOS/Windows)
# Download from: https://github.com/rust-lang/mdBook/releases
```

### Clone & Serve

```bash
git clone https://github.com/ZhangHanDong/harness-engineering-from-cc-to-ai-coding.git
cd harness-engineering-from-cc-to-ai-coding

# Build the book
mdbook build book

# Serve locally with live reload
mdbook serve book
# Default: http://localhost:3000
```

### Build Output

```
book/
├── src/           # Markdown source files
│   ├── SUMMARY.md # Table of contents / navigation
│   ├── assets/    # Images (cover, diagrams)
│   └── **/*.md    # Chapter files
└── book.toml      # mdBook configuration
```

---

## Book Structure

The book has **7 main parts** and **4 appendices**:

| Part | Topic |
|------|-------|
| Part 1 | Architecture — Claude Code's overall design, Agent Loop, tool orchestration |
| Part 2 | Prompt Engineering — System prompts, tool prompts, model-specific tuning |
| Part 3 | Context Management — Auto-compression, micro-compression, token budgets |
| Part 4 | Prompt Caching — Cache strategies, prompt cache design patterns |
| Part 5 | Security & Permissions — Permission modes, rule systems, YOLO classifier, Hooks |
| Part 6 | Advanced Subsystems — Multi-agent orchestration, skill systems, Feature Flags, unreleased pipelines |
| Part 7 | Lessons for AI Agent Builders — Production best practices, limitations, inspirations |

---

## Key Concepts Covered

### 1. Agent Loop Architecture

Claude Code's agent loop processes tasks through repeated cycles of:
1. **Observe** — Read context (files, shell output, user input)
2. **Plan** — Decide next action via LLM reasoning
3. **Act** — Execute tool calls (bash, file read/write, search)
4. **Reflect** — Incorporate results into next context window

**Key insight from the book:** The agent loop is not a simple REPL — it manages token budgets dynamically and compresses context when approaching limits.

### 2. Context Management Strategies

```
Context Window
├── System Prompt (static, cached)
├── Tool Definitions (static, cached)
├── Conversation History (dynamic)
│   ├── Full messages (recent)
│   └── Compressed summaries (older)  ← Auto-compression kicks in here
└── Current Task Buffer
```

**Auto-compression trigger:** When context approaches ~80% of max tokens, Claude Code summarizes older conversation turns to free space.

**Micro-compression:** Individual long tool outputs (e.g., large file reads, bash stdout) are truncated/summarized inline before being added to history.

### 3. Permission System

Claude Code uses a layered permission model:

```
Permission Levels (least → most privileged)
├── read-only       — File reads, directory listing, search
├── write           — File writes, creation, deletion
├── execute         — Shell commands (bash tool)
└── network         — HTTP requests, external services
```

**YOLO mode:** Bypasses interactive permission prompts. The book analyzes the YOLO classifier that determines which operations are auto-approved.

**Hooks system:** Pre/post action hooks allow custom permission logic, logging, or side effects to be injected into the tool execution pipeline.

### 4. Prompt Caching

Claude Code aggressively caches:
- **System prompt** — Rarely changes; always at top of cache
- **Tool definitions** — Static per session; cached after system prompt
- **File contents** — Frequently accessed files cached as pseudo-static context

**Pattern from the book:**
```
[CACHED BOUNDARY]
System Prompt
Tool Definitions
Common File Contents    ← Extended cache prefix
[DYNAMIC BOUNDARY]
Conversation turns      ← Not cached (changes each turn)
Current user message
```

### 5. Multi-Agent Orchestration

The book reveals Claude Code's sub-agent spawning for parallel tasks:

```
Orchestrator Agent
├── Sub-agent A: "Search codebase for X"
├── Sub-agent B: "Write tests for Y"
└── Sub-agent C: "Fix linting errors in Z"
        ↓
Results merged back into orchestrator context
```

---

## Applying Book Insights: Code Patterns

### Pattern 1: Implement a Token Budget Manager

Based on Part 3 (Context Management):

```python
class TokenBudgetManager:
    def __init__(self, max_tokens: int, compression_threshold: float = 0.8):
        self.max_tokens = max_tokens
        self.compression_threshold = compression_threshold
        self.history: list[dict] = []

    def should_compress(self, current_token_count: int) -> bool:
        return current_token_count / self.max_tokens > self.compression_threshold

    def compress_history(self, llm_client, history: list[dict]) -> list[dict]:
        """Summarize older turns to free token budget."""
        if len(history) <= 4:
            return history  # Keep at least 2 exchange pairs

        to_compress = history[:-4]  # Keep last 4 messages verbatim
        recent = history[-4:]

        summary_prompt = (
            "Summarize the following conversation history concisely, "
            "preserving key decisions, file paths, and technical details:\n\n"
            + "\n".join(f"{m['role']}: {m['content']}" for m in to_compress)
        )

        summary = llm_client.complete(summary_prompt)

        compressed = [{"role": "system", "content": f"[History Summary]: {summary}"}]
        return compressed + recent

    def add_message(self, message: dict, token_count: int, llm_client=None):
        self.history.append(message)
        if self.should_compress(token_count) and llm_client:
            self.history = self.compress_history(llm_client, self.history)
```

### Pattern 2: Layered Permission Gate

Based on Part 5 (Security & Permissions):

```python
from enum import IntEnum
from typing import Callable

class PermissionLevel(IntEnum):
    READ = 1
    WRITE = 2
    EXECUTE = 3
    NETWORK = 4

class PermissionGate:
    def __init__(self, auto_approve_level: PermissionLevel = PermissionLevel.READ):
        self.auto_approve_level = auto_approve_level
        self.hooks: list[Callable] = []

    def register_hook(self, hook: Callable):
        """Register a pre-execution hook (logging, auditing, etc.)"""
        self.hooks.append(hook)

    def request(self, action: str, level: PermissionLevel, details: dict) -> bool:
        # Run pre-execution hooks
        for hook in self.hooks:
            hook(action=action, level=level, details=details)

        # Auto-approve if within threshold
        if level <= self.auto_approve_level:
            return True

        # Interactive prompt for elevated permissions
        response = input(
            f"\n⚠️  Permission required: {action}\n"
            f"   Level: {level.name}\n"
            f"   Details: {details}\n"
            f"   Allow? [y/N]: "
        )
        return response.strip().lower() == "y"


# Usage
gate = PermissionGate(auto_approve_level=PermissionLevel.READ)

# Audit hook
def audit_log(action, level, details):
    print(f"[AUDIT] {action} ({level.name}): {details}")

gate.register_hook(audit_log)

# Auto-approved (READ <= READ threshold)
gate.request("read_file", PermissionLevel.READ, {"path": "src/main.py"})

# Prompts user (EXECUTE > READ threshold)
gate.request("run_bash", PermissionLevel.EXECUTE, {"command": "rm -rf /tmp/old"})
```

### Pattern 3: Prompt Cache Prefix Builder

Based on Part 4 (Prompt Caching):

```python
def build_cached_prompt_prefix(
    system_prompt: str,
    tool_definitions: list[dict],
    pinned_files: dict[str, str],  # {path: content}
) -> list[dict]:
    """
    Constructs the static (cacheable) portion of the context.
    Place this before dynamic conversation history.
    """
    messages = []

    # 1. System prompt — always first for maximum cache reuse
    messages.append({
        "role": "system",
        "content": system_prompt,
        "cache_control": {"type": "ephemeral"}  # Anthropic API cache hint
    })

    # 2. Tool definitions block
    if tool_definitions:
        tools_block = "# Available Tools\n" + "\n".join(
            f"## {t['name']}\n{t['description']}" for t in tool_definitions
        )
        messages.append({
            "role": "system",
            "content": tools_block,
            "cache_control": {"type": "ephemeral"}
        })

    # 3. Pinned file contents (frequently accessed files)
    for path, content in pinned_files.items():
        messages.append({
            "role": "system",
            "content": f"# File: {path}\n```\n{content}\n```",
            "cache_control": {"type": "ephemeral"}
        })

    return messages
```

---

## Contributing to the Book

The book source is Markdown files in `book/src/`. To contribute:

```bash
# 1. Fork and clone
git clone https://github.com/YOUR_USERNAME/harness-engineering-from-cc-to-ai-coding.git

# 2. Edit chapters
vim book/src/chapter-name.md

# 3. Update SUMMARY.md if adding new chapters
vim book/src/SUMMARY.md

# 4. Preview changes
mdbook serve book

# 5. Build to verify no errors
mdbook build book

# 6. Submit PR
```

### SUMMARY.md Format

```markdown
# Summary

[Introduction](./intro.md)

# Part 1: Architecture
- [Chapter 1: Agent Loop](./part1/agent-loop.md)
- [Chapter 2: Tool Orchestration](./part1/tools.md)

# Appendix
- [Appendix A: Glossary](./appendix/glossary.md)
```

---

## Troubleshooting

### mdBook not found

```bash
# Verify cargo installation
cargo --version

# Install mdbook
cargo install mdbook

# Add cargo bin to PATH if needed
export PATH="$HOME/.cargo/bin:$PATH"
```

### Build fails with missing file

```
error: Chapter 'X' not found at expected path 'src/...'
```

Check `book/src/SUMMARY.md` — every linked `.md` file must exist.

### Port 3000 already in use

```bash
mdbook serve book --port 3001
```

### Chinese characters display incorrectly

Ensure your terminal and browser use UTF-8 encoding. The book is UTF-8 throughout.

---

## Key Takeaways for AI Agent Builders

From **Part 7** of the book:

1. **Token budget is first-class** — Design your agent loop with token counting from day one, not as an afterthought
2. **Cache the static, compress the dynamic** — System prompts and tool definitions should never eat into your conversation budget
3. **Permission granularity matters** — Distinguish read/write/execute/network at the tool level, not just at the agent level
4. **Hooks over inheritance** — Claude Code uses hooks for extensibility; prefer composition over deep inheritance in agent frameworks
5. **Fail loud on context overflow** — Silent truncation causes subtle bugs; always surface context pressure to the operator
6. **Multi-agent for parallelism, single-agent for coherence** — Don't spawn sub-agents unless tasks are truly independent

---

## Resources

- **Book online:** https://zhanghandong.github.io/harness-engineering-from-cc-to-ai-coding/
- **English preview:** https://zhanghandong.github.io/harness-engineering-from-cc-to-ai-coding/en/
- **GitHub repo:** https://github.com/ZhangHanDong/harness-engineering-from-cc-to-ai-coding
- **mdBook docs:** https://rust-lang.github.io/mdBook/
- **Claude Code (analyzed version):** v2.1.88
- **License:** MIT
```
