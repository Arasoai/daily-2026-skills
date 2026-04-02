```markdown
---
name: claw-code-parity
description: Rust port parity work for the claw-code project, providing a harness runtime with agent tool orchestration capabilities
triggers:
  - "help me with claw-code parity"
  - "rust port for claw-code"
  - "set up claw code harness"
  - "how do I use the claw-code rust port"
  - "claw-code parity commands"
  - "run parity audit for claw-code"
  - "claw-code agent harness rust"
  - "port manifest for claw-code"
---

# Claw Code Parity (Rust Port)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

**claw-code-parity** is a temporary Rust port parity workspace for the [claw-code](https://github.com/instructkr/claw-code) project. It provides a faster, memory-safe harness runtime that mirrors the architectural patterns of Claude Code's agent harness — including tool wiring, command orchestration, and agent workflow management. This repo bridges the gap while the main `claw-code` repository completes its migration.

---

## What It Does

- Implements the core **agent harness runtime** in Rust for performance and memory safety
- Mirrors top-level subsystem names, command/tool inventories from the archived source
- Provides a **parity audit** mechanism to verify feature coverage against the original system
- Offers a CLI entrypoint for manifest output, subsystem listing, and parity summaries
- Designed to be orchestrated via AI coding agents (Claude Code, Codex, OmX, etc.)

---

## Installation & Setup

### Prerequisites

- Rust toolchain (stable, 1.75+): https://rustup.rs
- Cargo (bundled with Rust)

```bash
# Install Rust if needed
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Clone the repository
git clone https://github.com/ultraworkers/claw-code-parity.git
cd claw-code-parity

# Build the project
cargo build

# Build in release mode for production use
cargo build --release
```

---

## Key CLI Commands

The project exposes a CLI binary. Run via `cargo run --` or the compiled binary:

```bash
# Print parity summary
cargo run -- summary

# Print the current workspace manifest
cargo run -- manifest

# List subsystems (with optional limit)
cargo run -- subsystems --limit 16

# Run parity audit against local ignored archive (when present)
cargo run -- parity-audit

# Inspect mirrored command inventories
cargo run -- commands --limit 10

# Inspect mirrored tool inventories
cargo run -- tools --limit 10
```

After `cargo build --release`, use the binary directly:

```bash
./target/release/claw-code-parity summary
./target/release/claw-code-parity manifest
./target/release/claw-code-parity subsystems --limit 8
./target/release/claw-code-parity parity-audit
```

---

## Project Structure

```
.
├── src/
│   ├── main.rs           # CLI entrypoint and command dispatch
│   ├── commands.rs       # Command port metadata and registry
│   ├── tools.rs          # Tool port metadata and registry
│   ├── models.rs         # Core data structures (subsystems, modules, backlog)
│   ├── port_manifest.rs  # Workspace structure summary
│   └── query_engine.rs   # Renders parity summary from active workspace
├── tests/                # Integration and unit tests
├── Cargo.toml
└── README.md
```

---

## Core Data Models (Rust)

### Subsystem and Module Definitions

```rust
// src/models.rs

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct Subsystem {
    pub name: String,
    pub description: String,
    pub modules: Vec<Module>,
    pub status: PortStatus,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct Module {
    pub name: String,
    pub source_path: String,
    pub ported: bool,
    pub backlog_items: Vec<BacklogItem>,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct BacklogItem {
    pub id: String,
    pub description: String,
    pub priority: Priority,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub enum PortStatus {
    NotStarted,
    InProgress,
    Complete,
    Blocked(String),
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub enum Priority {
    High,
    Medium,
    Low,
}
```

### Command and Tool Metadata

```rust
// src/commands.rs

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct CommandEntry {
    pub name: String,
    pub description: String,
    pub args: Vec<ArgSpec>,
    pub ported: bool,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct ArgSpec {
    pub name: String,
    pub required: bool,
    pub arg_type: String,
}

pub fn command_registry() -> Vec<CommandEntry> {
    vec![
        CommandEntry {
            name: "summary".to_string(),
            description: "Render parity summary from active workspace".to_string(),
            args: vec![],
            ported: true,
        },
        CommandEntry {
            name: "manifest".to_string(),
            description: "Print current workspace manifest".to_string(),
            args: vec![],
            ported: true,
        },
        CommandEntry {
            name: "subsystems".to_string(),
            description: "List subsystems with optional limit".to_string(),
            args: vec![ArgSpec {
                name: "limit".to_string(),
                required: false,
                arg_type: "usize".to_string(),
            }],
            ported: true,
        },
        CommandEntry {
            name: "parity-audit".to_string(),
            description: "Run parity audit against local ignored archive".to_string(),
            args: vec![],
            ported: true,
        },
    ]
}
```

---

## CLI Entrypoint Pattern

```rust
// src/main.rs

use clap::{Parser, Subcommand};

#[derive(Parser, Debug)]
#[command(name = "claw-code-parity", version, about = "Claw Code Rust Port Parity Tool")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand, Debug)]
enum Commands {
    /// Render the parity summary
    Summary,
    /// Print the workspace manifest
    Manifest,
    /// List subsystems
    Subsystems {
        #[arg(long, default_value = "16")]
        limit: usize,
    },
    /// Run parity audit
    ParityAudit,
    /// List ported commands
    Commands {
        #[arg(long, default_value = "10")]
        limit: usize,
    },
    /// List ported tools
    Tools {
        #[arg(long, default_value = "10")]
        limit: usize,
    },
}

fn main() {
    let cli = Cli::parse();
    match cli.command {
        Commands::Summary => query_engine::render_summary(),
        Commands::Manifest => port_manifest::print_manifest(),
        Commands::Subsystems { limit } => {
            let subsystems = models::all_subsystems();
            for s in subsystems.iter().take(limit) {
                println!("{}: {} [{:?}]", s.name, s.description, s.status);
            }
        }
        Commands::ParityAudit => parity_audit::run(),
        Commands::Commands { limit } => {
            for cmd in commands::command_registry().iter().take(limit) {
                println!(
                    "{} — {} [ported: {}]",
                    cmd.name, cmd.description, cmd.ported
                );
            }
        }
        Commands::Tools { limit } => {
            for tool in tools::tool_registry().iter().take(limit) {
                println!(
                    "{} — {} [ported: {}]",
                    tool.name, tool.description, tool.ported
                );
            }
        }
    }
}
```

---

## Query Engine: Rendering a Parity Summary

```rust
// src/query_engine.rs

use crate::models::{all_subsystems, PortStatus};

pub fn render_summary() {
    let subsystems = all_subsystems();
    let total = subsystems.len();
    let complete = subsystems
        .iter()
        .filter(|s| matches!(s.status, PortStatus::Complete))
        .count();
    let in_progress = subsystems
        .iter()
        .filter(|s| matches!(s.status, PortStatus::InProgress))
        .count();

    println!("=== Claw Code Parity Summary ===");
    println!("Total subsystems : {}", total);
    println!("Complete         : {}", complete);
    println!("In Progress      : {}", in_progress);
    println!("Not Started      : {}", total - complete - in_progress);
    println!();
    println!("Subsystem Status:");
    for s in &subsystems {
        let status_str = match &s.status {
            PortStatus::Complete => "✅ Complete".to_string(),
            PortStatus::InProgress => "🔄 In Progress".to_string(),
            PortStatus::NotStarted => "⬜ Not Started".to_string(),
            PortStatus::Blocked(reason) => format!("🚫 Blocked: {}", reason),
        };
        println!("  {} — {}", s.name, status_str);
    }
}
```

---

## Running Tests

```bash
# Run all tests
cargo test

# Run tests with output
cargo test -- --nocapture

# Run a specific test
cargo test test_command_registry

# Run integration tests only
cargo test --test '*'
```

Example test:

```rust
// tests/test_commands.rs

#[cfg(test)]
mod tests {
    use claw_code_parity::commands::command_registry;

    #[test]
    fn test_command_registry_not_empty() {
        let cmds = command_registry();
        assert!(!cmds.is_empty(), "Command registry should not be empty");
    }

    #[test]
    fn test_summary_command_exists() {
        let cmds = command_registry();
        assert!(
            cmds.iter().any(|c| c.name == "summary"),
            "summary command must exist"
        );
    }

    #[test]
    fn test_all_required_commands_ported() {
        let required = ["summary", "manifest", "subsystems", "parity-audit"];
        let cmds = command_registry();
        for name in &required {
            let entry = cmds.iter().find(|c| &c.name == name);
            assert!(entry.is_some(), "Command '{}' must exist", name);
            assert!(entry.unwrap().ported, "Command '{}' must be ported", name);
        }
    }
}
```

---

## Cargo.toml Dependencies

```toml
[package]
name = "claw-code-parity"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "claw-code-parity"
path = "src/main.rs"

[dependencies]
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
thiserror = "1"

[dev-dependencies]
pretty_assertions = "1"
```

---

## Common Patterns

### Adding a New Subsystem

```rust
// In src/models.rs, extend all_subsystems()
pub fn all_subsystems() -> Vec<Subsystem> {
    vec![
        Subsystem {
            name: "tool-wiring".to_string(),
            description: "Tool registration and dispatch layer".to_string(),
            modules: vec![
                Module {
                    name: "tool_registry".to_string(),
                    source_path: "src/tools.rs".to_string(),
                    ported: true,
                    backlog_items: vec![],
                },
            ],
            status: PortStatus::InProgress,
        },
        // Add new subsystem here:
        Subsystem {
            name: "context-manager".to_string(),
            description: "Runtime context and state management".to_string(),
            modules: vec![],
            status: PortStatus::NotStarted,
        },
    ]
}
```

### Adding a New Tool

```rust
// src/tools.rs
pub fn tool_registry() -> Vec<ToolEntry> {
    vec![
        ToolEntry {
            name: "bash".to_string(),
            description: "Execute shell commands".to_string(),
            ported: true,
        },
        ToolEntry {
            name: "file-read".to_string(),
            description: "Read file contents".to_string(),
            ported: false,  // Mark as not yet ported
        },
        // Add new tool:
        ToolEntry {
            name: "web-search".to_string(),
            description: "Search the web for information".to_string(),
            ported: false,
        },
    ]
}
```

### Serializing State to JSON

```rust
use serde_json;
use crate::models::all_subsystems;

fn export_manifest_json() -> anyhow::Result<String> {
    let subsystems = all_subsystems();
    let json = serde_json::to_string_pretty(&subsystems)?;
    Ok(json)
}

// Usage
fn main() {
    match export_manifest_json() {
        Ok(json) => println!("{}", json),
        Err(e) => eprintln!("Failed to serialize manifest: {}", e),
    }
}
```

---

## Troubleshooting

### Build fails with missing features

```bash
# Ensure your Rust toolchain is up to date
rustup update stable
rustup default stable

# Clean and rebuild
cargo clean
cargo build
```

### `parity-audit` reports nothing

The parity audit requires the local ignored archive to be present. Ensure the archive path is configured (via environment variable if applicable):

```bash
export CLAW_ARCHIVE_PATH=/path/to/local/archive
cargo run -- parity-audit
```

### Subsystem list is empty

Check that `all_subsystems()` in `src/models.rs` returns a populated `Vec`. If extending the model, ensure the new entries are added before the closing `]`.

### Clap argument errors

Ensure argument names in the CLI match the `#[arg(long = "...")]` annotations exactly. Long flags use `--limit`, not `-limit`.

```bash
# Correct
cargo run -- subsystems --limit 8

# Incorrect (will error)
cargo run -- subsystems -limit 8
```

---

## Related Resources

- Main project: [instructkr/claw-code](https://github.com/instructkr/claw-code)
- Rust port branch: [`dev/rust`](https://github.com/instructkr/claw-code/tree/dev/rust)
- OmX orchestration tool: [Yeachan-Heo/oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex)
- Community Discord: [instruct.kr](https://instruct.kr/)
```
