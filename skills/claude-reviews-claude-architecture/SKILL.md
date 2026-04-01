```markdown
---
name: claude-reviews-claude-architecture
description: Skill for navigating and referencing the Claude Code architectural analysis — 17-part deep dive written by Claude about its own source code
triggers:
  - "explain claude code architecture"
  - "how does claude code work internally"
  - "claude code query engine explanation"
  - "how does claude code handle tools"
  - "claude code multi-agent coordination"
  - "claude code permission system"
  - "claude code context management"
  - "read the claude code architecture analysis"
---

# Claude Reviews Claude — Architecture Analysis Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

**claude-reviews-claude** is a 17-part architectural analysis of Claude Code (v2.1.88) written *by Claude itself* after reading 1,902 files and 477,439 lines of TypeScript source code. It is not a library or CLI — it is a structured reference document covering every major subsystem of Claude Code.

Use this skill when a developer asks about Claude Code internals, wants to understand design patterns used in Claude Code, or needs to navigate the analysis documents.

---

## Repository Layout

```
architecture/
├── zh-CN/                        # Chinese versions (primary)
│   ├── 00-overview.md            # Full system map, 17 subsystems
│   ├── 01-query-engine.md        # Core LLM loop (1296 lines)
│   ├── 02-tool-system.md         # 42+ tools, schema-driven registry
│   ├── 03-coordinator.md         # Parallel worker orchestration
│   ├── 04-plugin-system.md       # Plugin load/validate/integrate
│   ├── 05-hook-system.md         # PreToolUse / PostToolUse hooks
│   ├── 06-bash-engine.md         # Sandboxed command execution
│   ├── 07-permission-pipeline.md # 7-layer defense-in-depth
│   ├── 08-agent-swarms.md        # Multi-agent mailbox IPC
│   ├── 09-session-persistence.md # Append-only JSONL storage
│   ├── 10-context-assembly.md    # 3-layer context construction
│   ├── 11-compact-system.md      # 4-tier compression cascade
│   ├── 12-startup-bootstrap.md   # Fast-path init, dynamic imports
│   ├── 13-bridge-system.md       # Remote control / IDE bridge
│   ├── 14-ui-state-management.md # Ink + React 19, 140+ components
│   ├── 15-services-api-layer.md  # API client, MCP, OAuth
│   ├── 16-infrastructure-config.md # Settings, feature flags, telemetry
│   └── 17-telemetry-privacy-operations.md # Dual-channel telemetry
└── (English mirrors at same filenames, root level)
```

---

## The Six Architectural Pillars

### 1. Query Engine (`01-query-engine.md`)
The core `while(true)` loop that drives everything:

```
User Input
  → QueryEngine.query()
  → Claude API (streaming)
  → stop_reason = end_turn?  → output
  → stop_reason = tool_use?
      → Permission Check
      → Execute Tool
      → Inject Result
      → loop
```

Key facts:
- 1,296 lines, 12-step state machine
- Manages LLM queries, tool loops, session state
- All intelligence lives in the LLM; the scaffold is just a loop

### 2. Tool System (`02-tool-system.md`)
42+ tools registered as self-contained modules:

```typescript
// Each tool implements this contract (~30 methods)
interface Tool {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  execute(input: unknown, context: ToolContext): Promise<ToolResult>;
  isEnabled(): boolean;
  userFacingName(): string;
}
```

Tool categories:
- File I/O: `Read`, `Write`, `Edit`, `MultiEdit`
- Shell: `Bash` (sandboxed)
- Search: `Grep`, `Glob`, `WebSearch`
- Agent: `Task`, `SubAgent`
- MCP: dynamically registered external tools

### 3. Permission Pipeline (`07-permission-pipeline.md`)
7-layer defense-in-depth:

```
Layer 1: Config rules (allowlist/denylist patterns)
Layer 2: Tool-level checks
Layer 3: AST-based command analysis
Layer 4: YOLO classifier (risk scoring)
Layer 5: User confirmation prompts
Layer 6: OS sandbox (macOS Seatbelt / Linux seccomp)
Layer 7: Audit logging
```

### 4. Context Assembly & Compression (`10`, `11`)
4-tier cascade for 200K context window:

```
Tier 1: Micro-compression  (inline dedup, whitespace)
Tier 2: Truncation         (drop oldest non-pinned turns)
Tier 3: LLM summarization  (AI-generated session summary)
Tier 4: Emergency compact  (nuclear option, keep system prompt only)
```

### 5. Multi-Agent Swarms (`08-agent-swarms.md`)
Three backends, mailbox IPC:

```
Backends:
  - iTerm2  (macOS terminal panes)
  - tmux    (cross-platform multiplexer)
  - In-process (SubAgent tool, no shell)

Coordination:
  - Parent assigns task → writes to worker mailbox
  - Worker polls mailbox → executes → writes result
  - Parent aggregates results → continues loop
```

7 task types: file editing, code search, test running, web search, sub-planning, validation, summarization.

### 6. Terminal UI (`14-ui-state-management.md`)
- Forked [Ink](https://github.com/vadimdemedes/ink) + React 19
- 140+ components
- Vim keybinding mode
- Computer Use support
- IDE bridge via `13-bridge-system.md` (polling dispatch loop, crash recovery)

---

## Session Persistence (`09-session-persistence.md`)

```
~/.claude/projects/<hash>/
  └── <session-uuid>.jsonl   # append-only, one JSON object per line

Each line:
{
  "type": "message" | "tool_use" | "tool_result" | "summary",
  "uuid": "...",
  "parent_uuid": "...",      # linked list for replay
  "timestamp": "...",
  "content": { ... }
}

Resume: read last 64KB → reconstruct minimal context → continue
```

---

## Startup & Bootstrap (`12-startup-bootstrap.md`)

Fast-path cascade:
1. Parse CLI args (no config read yet)
2. Detect `--print` / `--dangerously-skip-permissions` flags → skip heavy init
3. Dynamic `import()` of subsystems (avoids paying startup cost upfront)
4. Pre-connect to Anthropic API (parallel with config load)
5. Initialize global state singletons
6. Render first UI frame

---

## Hook System (`05-hook-system.md`)

```typescript
// Three hook points
type HookPoint =
  | "PreToolUse"    // before any tool executes
  | "PostToolUse"   // after tool result available
  | "SessionStart"; // once per session init

// Hook registration (in settings or plugin)
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash", "command": "echo 'bash called'" }
    ]
  }
}
```

Hooks receive full tool input/output via stdin as JSON and can block execution by exiting non-zero.

---

## Plugin System (`04-plugin-system.md`)

6 plugin sources, loaded in priority order:
1. Enterprise policy (highest, cannot be overridden)
2. User global config (`~/.claude/settings.json`)
3. Project config (`.claude/settings.json`)
4. CLAUDE.md memory files (per-directory)
5. MCP servers (dynamic tool injection)
6. CLI flags (session-scoped overrides)

---

## Source Code References

The analysis is based on Claude Code v2.1.88. Community-restored source:

| Repo | Notes |
|------|-------|
| `instructkr/claw-code` | Restored TypeScript source |
| `ChinaSiro/claude-code-sourcemap` | Extracted from source maps |

---

## Navigation Guide for AI Agents

| Developer question | Point to |
|--------------------|----------|
| "How does Claude Code call tools?" | `01-query-engine.md` + `02-tool-system.md` |
| "Why did Claude ask for permission?" | `07-permission-pipeline.md` |
| "How do I add a hook?" | `05-hook-system.md` |
| "How does session resume work?" | `09-session-persistence.md` |
| "How do I use Claude Code with multiple agents?" | `08-agent-swarms.md` + `03-coordinator.md` |
| "Why is context getting truncated?" | `11-compact-system.md` |
| "How does CLAUDE.md work?" | `10-context-assembly.md` |
| "How do I write a plugin/extension?" | `04-plugin-system.md` |
| "How does the IDE integration work?" | `13-bridge-system.md` |
| "What telemetry does Claude Code collect?" | `17-telemetry-privacy-operations.md` |
| "How does Claude Code start up so fast?" | `12-startup-bootstrap.md` |

---

## Key Design Patterns (Reusable)

1. **Dumb loop, smart model** — Keep orchestration simple; push intelligence into the LLM
2. **Schema-driven tool registry** — Tools self-describe via JSON Schema; no manual dispatch tables
3. **Append-only event log** — Session state is a JSONL log, not mutable DB rows
4. **Layered permission defense** — Static rules → dynamic analysis → user prompt → OS enforcement
5. **4-tier context compression** — Graceful degradation instead of hard context-exceeded errors
6. **Mailbox IPC for agents** — Filesystem-based message passing; survives process crashes

---

## Cloning and Reading Locally

```bash
git clone https://github.com/openedclaude/claude-reviews-claude.git
cd claude-reviews-claude

# English analysis
ls architecture/*.md

# Chinese analysis
ls architecture/zh-CN/*.md

# Start with the overview
open architecture/zh-CN/00-overview.md
# or
cat architecture/00-overview.md
```

No build step, no dependencies. Pure Markdown.

---

## Troubleshooting

**"I can't find the English version of a specific episode"**
→ English files live at `architecture/<NN>-<name>.md` (root level).
→ Chinese files live at `architecture/zh-CN/<NN>-<name>.md`.

**"The source code links don't work"**
→ This repo contains analysis only, not the source. See `instructkr/claw-code` or `ChinaSiro/claude-code-sourcemap` for source.

**"Episode numbers don't match the table of contents"**
→ Episodes are 0-indexed (00-overview through 17-telemetry). The roadmap section lists them 1-indexed in prose — Episode 1 = file `01-query-engine.md`.

**"I want the latest Claude Code analysis"**
→ Season 1 covers v2.1.88 and is complete (17 episodes). Watch the repo for Season 2 announcements.
```
