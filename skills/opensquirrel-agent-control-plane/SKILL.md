```markdown
---
name: opensquirrel-agent-control-plane
description: Native Rust/GPUI control plane for running multiple AI coding agents (Claude Code, Codex, Cursor, OpenCode) side by side with coordinator/worker delegation, SSH targeting, and persistent sessions.
triggers:
  - run multiple AI agents side by side
  - opensquirrel setup and configuration
  - multi-agent grid with claude and codex
  - coordinate AI coding agents with opensquirrel
  - run claude code and codex simultaneously
  - SSH remote machine targeting for agents
  - persistent agent sessions with opensquirrel
  - delegate tasks between AI coding agents
---

# OpenSquirrel Agent Control Plane

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

OpenSquirrel is a native macOS application (Rust + GPUI, Metal-rendered, no Electron) that acts as a control plane for running Claude Code, Codex, Cursor, and OpenCode simultaneously. It supports automatic sub-agent delegation, remote SSH targeting, MCP server integration, and persistent multi-turn sessions.

## Installation & Build

**Prerequisites:** Rust toolchain, macOS with Metal GPU support.

```bash
git clone https://github.com/Infatoshi/OpenSquirrel
cd OpenSquirrel
cargo build --release
./target/release/opensquirrel
```

**As a macOS .app bundle:**

```bash
cargo build --release
mkdir -p dist/OpenSquirrel.app/Contents/{MacOS,Resources}
cp target/release/opensquirrel dist/OpenSquirrel.app/Contents/MacOS/OpenSquirrel
cp assets/OpenSquirrel.icns dist/OpenSquirrel.app/Contents/Resources/
open dist/OpenSquirrel.app
```

> **Note:** The `.app` bundle does not inherit your shell PATH. If `claude`, `npx`, or other agent CLIs aren't found, run the binary directly: `./target/release/opensquirrel`

## Configuration

Config file lives at `~/.opensquirrel/config.toml`. State (agents, transcripts, scroll positions) is persisted at `~/.opensquirrel/state.json`.

### Full config.toml example

```toml
[settings]
theme = "midnight"          # midnight | charcoal | gruvbox | solarized-dark | light | solarized-light | ops | monokai-pro
font_size = 14
compact_context = false

[[runtimes]]
name = "claude"
binary = "claude"           # must be on PATH when running the binary directly
mode = "persistent"         # persistent stdin (multi-turn)
permission_flags = ["--dangerously-skip-permissions"]

[[runtimes]]
name = "codex"
binary = "npx"
args = ["codex"]
mode = "oneshot"
permission_flags = ["--dangerously-bypass-approvals-and-sandbox"]

[[runtimes]]
name = "cursor"
binary = "cursor-agent"
mode = "oneshot"
permission_flags = ["--yolo"]

[[runtimes]]
name = "opencode"
binary = "opencode"
mode = "oneshot"            # auto-approved in run mode

[[models]]
runtime = "claude"
model = "claude-opus-4-5"   # coordinator/primary
alias = "opus"

[[models]]
runtime = "claude"
model = "claude-sonnet-4-5" # worker/sub-agent
alias = "sonnet"

[[mcp_servers]]
name = "playwright"
args = ["npx", "@playwright/mcp@latest"]

[[mcp_servers]]
name = "browser-use"
args = ["uvx", "browser-use-mcp"]

[[machines]]
name = "local"
host = "localhost"

[[machines]]
name = "dev-box"
host = "192.168.1.100"
user = "ubuntu"
ssh_key = "~/.ssh/id_ed25519"
tmux_session = "opensquirrel"
```

## Supported Runtimes

| Runtime | Mode | Permission bypass flag |
|---------|------|------------------------|
| Claude Code | Persistent stdin (multi-turn) | `--dangerously-skip-permissions` |
| Codex | One-shot per prompt | `--dangerously-bypass-approvals-and-sandbox` |
| Cursor Agent | One-shot per prompt | `--yolo` |
| OpenCode | One-shot per prompt | Auto-approved in `run` mode |

## Keybinds Reference

| Key | Action |
|-----|--------|
| `Enter` | Send prompt to focused agent |
| `Escape` | Dismiss overlay (palette, setup wizard, search) |
| `Cmd-N` | New agent — opens setup wizard |
| `Cmd-K` | Command palette (themes, compact context, kill agent, views) |
| `Cmd-F` | Search across all agent transcripts |
| `Cmd-]` / `Cmd-[` | Next / prev pane within group |
| `Cmd-}` / `Cmd-{` | Next / prev group |
| `Cmd-V` | Paste clipboard into prompt |
| `Cmd-=` / `Cmd--` | Zoom in / out |

**Setup wizard navigation:** Arrow keys to move, `Enter` to drill into directories, `Backspace` to go up, `Tab` to advance step, `Shift-Tab` to go back.

## Architecture Overview

```
~7,200 lines of Rust across 3 files:
src/
  main.rs          — UI, agent lifecycle, rendering, keybinds, persistence
  lib.rs           — Line classification, markdown parsing, diff summarization, helpers
tests/
  state_tests.rs   — 30 integration tests (navigation, scrolling, themes, search, agent lifecycle)
```

Built on [GPUI](https://crates.io/crates/gpui) — the GPU-rendered UI framework from Zed, used as a standalone crate. Rendered via Metal on macOS.

## Agent Layout Behavior

Agents auto-arrange based on count:

| Agent count | Layout |
|-------------|--------|
| 1 | Full screen |
| 2 | Side-by-side split |
| 3 | 2+1 arrangement |
| 4 | 2×2 grid |
| 5+ | Dynamic tiling |

## Coordinator / Worker Delegation Pattern

A primary "coordinator" agent (typically Claude Opus) can delegate focused sub-tasks to worker agents (typically Claude Sonnet). Workers return condensed results, not full transcripts.

**Typical setup:**
1. `Cmd-N` → Select runtime: `claude`, model: `opus` → this becomes your coordinator
2. `Cmd-N` → Select runtime: `claude`, model: `sonnet` → worker agent
3. In the coordinator pane, prompt it to delegate: _"Spawn a worker to refactor the auth module and report back"_

The coordinator issues structured sub-prompts; workers execute and summarize.

## Remote Machine Targeting via SSH

Configure machines in `config.toml`, then select a machine in the setup wizard when creating a new agent. OpenSquirrel connects via SSH + tmux.

```toml
[[machines]]
name = "gpu-server"
host = "10.0.0.50"
user = "ubuntu"
ssh_key = "~/.ssh/id_ed25519"
tmux_session = "opensquirrel-agents"
```

The agent CLI will run inside a tmux session on the remote host. Ensure the agent binary (`claude`, `npx codex`, etc.) is installed on the remote machine and available on its PATH.

## MCP Server Integration

MCP servers are passed directly as CLI args to agent runtimes. Select which MCP servers to attach per agent during the setup wizard.

```toml
[[mcp_servers]]
name = "playwright"
args = ["npx", "@playwright/mcp@latest"]

[[mcp_servers]]
name = "browser-use"
args = ["uvx", "browser-use-mcp"]
```

When an agent is created with Playwright attached, OpenSquirrel appends the MCP args to the agent's launch command automatically.

## Persistent Sessions

Agent state survives app restarts:
- Transcripts and scroll positions stored in `~/.opensquirrel/state.json`
- Pending (unsent) prompts are preserved
- Interrupted multi-turn sessions (Claude Code persistent mode) can be resumed

To manually clear state:

```bash
rm ~/.opensquirrel/state.json
```

## Transcript Search

`Cmd-F` opens a search overlay that queries across all agent transcripts simultaneously. Results are highlighted inline in each pane. Press `Escape` to dismiss.

## Themes

Available themes: `midnight`, `charcoal`, `gruvbox`, `solarized-dark`, `light`, `solarized-light`, `ops`, `monokai-pro`

Change via `Cmd-K` → Command Palette → "Theme: ..." or set in `config.toml`:

```toml
[settings]
theme = "gruvbox"
```

## Running the Test Suite

```bash
cargo test
```

Tests in `tests/state_tests.rs` cover: pane navigation, scroll behavior, theme switching, transcript search, agent lifecycle (spawn, kill, resume).

## Common Patterns

### Start a 4-agent parallel session

```
1. cargo build --release && ./target/release/opensquirrel
2. Cmd-N → Claude (Opus) on local machine, no MCP  [coordinator]
3. Cmd-N → Claude (Sonnet) on local machine         [worker: backend]
4. Cmd-N → Codex on local machine                   [worker: frontend]
5. Cmd-N → OpenCode on local machine                [worker: tests]
```

Agents auto-arrange into a 2×2 grid. Focus coordinator pane and send a high-level task.

### Target a remote GPU machine

```
1. Add machine to ~/.opensquirrel/config.toml
2. Cmd-N → select runtime → select machine: "gpu-server"
3. Agent runs inside tmux on the remote host
```

### Attach a browser automation MCP server

```
1. Ensure [[mcp_servers]] entry exists in config.toml
2. Cmd-N → setup wizard → step "MCP Servers" → check "playwright"
3. Agent launched with Playwright MCP wired in
```

### Compact context to reduce token usage

`Cmd-K` → "Compact Context" — summarizes current transcript in-place for the focused agent, reducing context window usage while preserving key decisions.

## Troubleshooting

**Agent binary not found (`claude`, `npx`, `codex` not on PATH)**
- Run `./target/release/opensquirrel` directly instead of via `.app` bundle
- Or set full paths in `config.toml`: `binary = "/usr/local/bin/claude"`

**Remote SSH agent not connecting**
- Test SSH manually: `ssh -i ~/.ssh/id_ed25519 ubuntu@10.0.0.50`
- Ensure `tmux` is installed on the remote host
- Verify agent CLI is on the remote machine's PATH

**State file corruption after crash**
```bash
rm ~/.opensquirrel/state.json
# OpenSquirrel will create a fresh state on next launch
```

**Metal/GPU errors on startup**
- Requires macOS with Metal support (macOS 10.14+, any Apple Silicon or Intel Mac with discrete/integrated GPU)
- OpenSquirrel does not support Linux or Windows

**Build failures**
```bash
rustup update stable
cargo clean
cargo build --release
```

**Agents not receiving prompts in persistent mode**
- Claude Code persistent mode uses stdin; ensure `--dangerously-skip-permissions` is set in config so it doesn't pause for approval prompts mid-session

## Environment Variables

OpenSquirrel itself does not read API keys — those are consumed by the agent CLIs it launches. Ensure these are set in your shell environment before launching the binary:

```bash
export ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY        # for Claude Code
export OPENAI_API_KEY=$OPENAI_API_KEY              # for Codex
# Cursor Agent uses its own auth; log in via cursor CLI first
```
```
