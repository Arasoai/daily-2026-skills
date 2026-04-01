```markdown
---
name: claurst-claude-code-rust
description: Clean-room Rust reimplementation of Claude Code CLI with multi-agent orchestration, tool system, and memory consolidation
triggers:
  - "use claurst"
  - "build with claurst"
  - "claude code rust implementation"
  - "set up claurst CLI"
  - "claurst tool system"
  - "claurst agent orchestration"
  - "implement claurst memory"
  - "claurst bash tool"
---

# Claurst — Claude Code in Rust

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Claurst is a clean-room Rust reimplementation of Claude Code's behavior, derived from exhaustive behavioral specifications (not the original TypeScript source). It reproduces the full Claude Code CLI feature set — multi-agent orchestration, 40+ tools, background memory consolidation ("dream"), KAIROS proactive mode, ULTRAPLAN remote planning, and the BUDDY companion system — in idiomatic Rust.

---

## Project Structure

```
claurst/
├── spec/                        # Behavioral specifications (AI-generated from analysis)
├── src-rust/
│   └── crates/
│       ├── cli/                 # Entry point (main.rs)
│       ├── assistant/           # Core agent loop, KAIROS proactive mode
│       ├── tools/               # All tool implementations (bash, fs, search, etc.)
│       ├── query/               # Memory compaction (autoDream / consolidationPrompt)
│       ├── buddy/               # Tamagotchi companion (feature-flagged)
│       └── ...
```

---

## Installation

### Prerequisites

- Rust toolchain (stable, 1.75+): https://rustup.rs
- An Anthropic API key set as `ANTHROPIC_API_KEY`

### Build from Source

```bash
git clone https://github.com/Kuberwastaken/claurst
cd claurst/src-rust

# Default build
cargo build --release

# With optional features
cargo build --release --features buddy,proactive

# Run directly
cargo run --release -- --help
```

### Install Binary

```bash
cargo install --path src-rust/crates/cli
claurst --help
```

---

## Key CLI Commands

```bash
# Start interactive session
claurst

# Run a single prompt non-interactively
claurst -p "Refactor src/lib.rs to use async/await"

# Specify model
claurst --model claude-opus-4-6

# Enable verbose tool output
claurst --verbose

# Use a specific working directory
claurst --cwd /path/to/project

# Trigger ULTRAPLAN mode for complex tasks
claurst --ultraplan "Design the full architecture for a distributed cache"

# Show version
claurst --version
```

---

## Configuration

Configuration is resolved in this order (later entries override earlier):

1. `~/.claurst/config.toml` — global user config
2. `.claurst.toml` in the project root — project-level config
3. Environment variables — highest priority

### `~/.claurst/config.toml`

```toml
[api]
# Key is read from ANTHROPIC_API_KEY env var by default
model = "claude-opus-4-6"
max_tokens = 8192

[memory]
memory_dir = "~/.claurst/memory"
auto_dream = true
dream_interval_hours = 24
dream_session_threshold = 5

[tools]
allowed = ["bash", "read_file", "write_file", "search", "grep"]
bash_timeout_secs = 30

[kairos]
enabled = false          # Requires PROACTIVE feature flag at compile time
tick_interval_secs = 60
blocking_budget_secs = 15

[ultraplan]
enabled = true
poll_interval_secs = 3
max_duration_secs = 1800  # 30 minutes
model = "claude-opus-4-6"
```

### Environment Variables

```bash
export ANTHROPIC_API_KEY="sk-ant-..."          # Required
export CLAURST_MODEL="claude-sonnet-4-5"       # Override model
export CLAURST_MEMORY_DIR="$HOME/.claurst/mem" # Memory storage path
export CLAURST_LOG_LEVEL="debug"               # trace|debug|info|warn|error
export CLAURST_MAX_TOKENS="4096"               # Token budget
export CLAURST_BASH_TIMEOUT="60"               # Bash tool timeout
```

---

## Core Concepts

### Agent Loop

The main loop in `crates/assistant/` runs a read-eval-act cycle:

1. Receive user message
2. Build context (memory, tool results, conversation history)
3. Call Anthropic API
4. Parse tool calls from response
5. Execute tools, append results
6. Continue until no more tool calls or stop condition

### Tool System

Tools implement the `Tool` trait:

```rust
// crates/tools/src/lib.rs
use async_trait::async_trait;
use serde_json::Value;

#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &'static str;
    fn description(&self) -> &'static str;
    fn input_schema(&self) -> Value;
    async fn execute(&self, input: Value) -> ToolResult;
}

#[derive(Debug)]
pub struct ToolResult {
    pub output: String,
    pub is_error: bool,
}
```

### Registering a Custom Tool

```rust
// crates/tools/src/custom_tool.rs
use async_trait::async_trait;
use serde_json::{json, Value};
use crate::{Tool, ToolResult};

pub struct MyCustomTool;

#[async_trait]
impl Tool for MyCustomTool {
    fn name(&self) -> &'static str { "my_custom_tool" }

    fn description(&self) -> &'static str {
        "Does something useful for the agent"
    }

    fn input_schema(&self) -> Value {
        json!({
            "type": "object",
            "properties": {
                "target": {
                    "type": "string",
                    "description": "The target to operate on"
                }
            },
            "required": ["target"]
        })
    }

    async fn execute(&self, input: Value) -> ToolResult {
        let target = input["target"].as_str().unwrap_or("");
        ToolResult {
            output: format!("Processed: {}", target),
            is_error: false,
        }
    }
}

// Register in tool registry (crates/cli/src/main.rs)
registry.register(Box::new(MyCustomTool));
```

---

## Built-in Tools Reference

| Tool | Description |
|------|-------------|
| `bash` | Execute shell commands with timeout |
| `read_file` | Read file contents |
| `write_file` | Write or overwrite a file |
| `edit_file` | Apply targeted edits to a file |
| `list_dir` | List directory contents |
| `search_files` | Glob-based file search |
| `grep` | Regex search across files |
| `web_fetch` | Fetch a URL |
| `send_user_file` | Push file to user (KAIROS only) |
| `subscribe_pr` | Monitor pull request (KAIROS only) |

---

## Memory System (autoDream)

The dream system consolidates conversation memory in the background.

### Three-Gate Trigger

All three conditions must be true before a dream runs:

```rust
// Pseudo-representation of gate logic
fn should_dream(state: &DreamState) -> bool {
    let hours_since = state.last_dream.elapsed().as_secs() / 3600;
    let sessions_since = state.sessions_since_last_dream;
    let lock_available = state.consolidation_lock.try_acquire().is_ok();

    hours_since >= 24 && sessions_since >= 5 && lock_available
}
```

### Four Dream Phases

Defined in `crates/query/src/compact.rs`:

1. **Orient** — `ls` memory dir, read `MEMORY.md`, skim topic files
2. **Gather** — Pull recent session logs, extract key facts and decisions
3. **Synthesize** — Merge into structured topic files, resolve contradictions
4. **Commit** — Write updated files, release lock, record timestamp

### Memory Directory Layout

```
~/.claurst/memory/
├── MEMORY.md          # Top-level summary, always read first
├── topics/
│   ├── architecture.md
│   ├── preferences.md
│   └── projects.md
└── sessions/
    ├── 2026-03-31.log
    └── 2026-04-01.log
```

### Manually Trigger a Dream

```bash
claurst --dream-now
```

---

## BUDDY — Tamagotchi Companion (Feature Flag)

Build with `--features buddy` to enable.

```bash
cargo build --release --features buddy
```

Species is deterministically assigned per user via Mulberry32 PRNG seeded from `userId` hash + salt `friend-2026-401`. Same user always gets the same species.

```rust
// Mulberry32 PRNG (ported to Rust in crates/buddy/src/rng.rs)
fn mulberry32(seed: u32) -> impl FnMut() -> f64 {
    let mut s = seed;
    move || {
        s = s.wrapping_add(0x6D2B79F5);
        let mut t = (s ^ (s >> 15)).wrapping_mul(1u32.wrapping_add(s));
        t = (t ^ (t >> 7)).wrapping_mul(61u32.wrapping_add(t)) ^ t;
        ((t ^ (t >> 14)) as f64) / 4294967296.0
    }
}
```

### Species Rarity Table

| Rarity | Chance | Species |
|--------|--------|---------|
| Common | 60% | Pebblecrab, Dustbunny, Mossfrog, Twigling, Dewdrop, Puddlefish |
| Uncommon | 25% | Cloudferret, Gustowl, Bramblebear, Thornfox |
| Rare | 10% | Crystaldrake, Deepstag, Lavapup |
| Epic | 4% | Stormwyrm, Voidcat, Aetherling |
| Legendary | 1% | Cosmoshale, Nebulynx |

Shiny chance: independent 1%. Shiny Legendary = 0.01%.

---

## ULTRAPLAN — Remote 30-Minute Planning

For complex architectural tasks, ULTRAPLAN offloads planning to a remote Cloud Container Runtime (CCR) session:

```bash
claurst --ultraplan "Design a horizontally scalable job queue with exactly-once delivery"
```

Flow:
1. Task sent to remote CCR running `tengu_ultraplan_model` (Opus 4.6)
2. Local terminal polls every 3 seconds
3. Browser UI opens for live monitoring and approval
4. On approval, `__ULTRAPLAN_TELEPORT_LOCAL__` sentinel returns result to terminal

Configure timeout and poll interval in `config.toml`:

```toml
[ultraplan]
poll_interval_secs = 3
max_duration_secs = 1800
```

---

## KAIROS — Proactive Mode (Feature Flag)

Build with `--features proactive`:

```bash
cargo build --release --features proactive
claurst --kairos
```

KAIROS maintains append-only daily logs and receives `<tick>` prompts on an interval to decide whether to act proactively. Actions taking >15 seconds are deferred.

```toml
[kairos]
enabled = true
tick_interval_secs = 60
blocking_budget_secs = 15
```

---

## Multi-Agent Orchestration

Spawn subagents from within a tool or agent context:

```rust
use claurst_assistant::{Agent, AgentConfig, Message};

let subagent = Agent::new(AgentConfig {
    model: "claude-sonnet-4-5".into(),
    system_prompt: "You are a focused code reviewer.".into(),
    tools: vec!["read_file", "grep"],
    max_tokens: 2048,
    ..Default::default()
});

let result = subagent
    .run(vec![Message::user("Review src/lib.rs for unsafe blocks")])
    .await?;

println!("{}", result.final_response());
```

---

## Common Patterns

### Non-Interactive Scripting

```bash
#!/usr/bin/env bash
# Run claurst headlessly in CI
export ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY"

claurst -p "Run the test suite and summarize failures" \
  --no-interactive \
  --output-format json \
  > results.json
```

### Piping Context

```bash
# Feed file contents as context
cat src/main.rs | claurst -p "Explain this code and suggest improvements"

# Pipe git diff for review
git diff HEAD~1 | claurst -p "Review this diff for bugs"
```

### Custom System Prompt

```bash
claurst --system "You are an expert in embedded Rust. Always prefer no_std solutions." \
  -p "Implement a ring buffer for a bare-metal target"
```

---

## Troubleshooting

### API Key Not Found

```
Error: ANTHROPIC_API_KEY not set
```

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# Or add to ~/.claurst/config.toml under [api]
```

### Bash Tool Timeout

```
ToolError: bash execution exceeded 30s timeout
```

Increase timeout:
```toml
[tools]
bash_timeout_secs = 120
```

Or per-invocation:
```bash
claurst --bash-timeout 120 -p "Run the full integration test suite"
```

### Dream Not Running

Check that all three gates are satisfied:
- 24h have passed since last dream
- 5+ sessions since last dream
- No concurrent dream is running (check for stale lock: `~/.claurst/memory/.dream.lock`)

Force a dream manually:
```bash
claurst --dream-now
rm ~/.claurst/memory/.dream.lock  # If lock is stale
```

### Build Errors with Feature Flags

```
error[E0432]: unresolved import `claurst_buddy`
```

Feature flags must be passed consistently:
```bash
cargo build --features buddy,proactive
# Not just: cargo build
```

### Memory Growing Too Large

```bash
# Inspect memory directory size
du -sh ~/.claurst/memory/

# Manually compact (triggers dream immediately)
claurst --dream-now --verbose
```

---

## Spec-Driven Development

To contribute or extend claurst, follow the two-phase clean-room process:

1. **Spec phase** (`spec/`): Write behavioral specifications — inputs, outputs, tool contracts, data flows. No implementation details.
2. **Implementation phase** (`src-rust/`): Implement from spec alone. Never reference external proprietary source.

This mirrors the Phoenix Technologies v. IBM (1984) clean-room BIOS precedent.

---

## Resources

- **Repository**: https://github.com/Kuberwastaken/claurst
- **Blog breakdown**: https://kuber.studio/blog/AI/Claude-Code's-Entire-Source-Code-Got-Leaked-via-a-Sourcemap-in-npm,-Let's-Talk-About-it
- **Spec directory**: `spec/` — full behavioral specifications
- **Anthropic API docs**: https://docs.anthropic.com
```
