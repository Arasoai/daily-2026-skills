```markdown
---
name: rtk-token-optimizer
description: CLI proxy that reduces LLM token consumption by 60-90% on common dev commands using a single Rust binary
triggers:
  - reduce token usage with rtk
  - install rtk for claude code
  - rtk token savings setup
  - optimize llm context with rtk
  - rtk git status compact output
  - set up rtk hook for ai coding
  - rtk command not working
  - how to use rtk with cursor or windsurf
---

# RTK Token Optimizer

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

RTK is a high-performance CLI proxy written in Rust that intercepts common dev commands (`git`, `ls`, `cargo test`, `pytest`, etc.) and compresses their output before it reaches your LLM context window. A single static binary with zero runtime dependencies adds less than 10ms overhead while cutting token usage by 60–90%.

---

## Installation

### Homebrew (recommended)
```bash
brew install rtk
```

### Quick install (Linux/macOS)
```bash
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh
# Add to PATH if needed:
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

### Cargo (from source)
```bash
cargo install --git https://github.com/rtk-ai/rtk
```

### Verify
```bash
rtk --version   # rtk 0.28.2
rtk gain        # shows token savings stats
```

> **Crates.io collision**: A different package called "rtk" exists on crates.io. Always use `--git` with cargo.

---

## Hook Setup (Auto-Rewrite)

The hook transparently rewrites Bash tool calls (e.g., `git status` → `rtk git status`) before execution. Claude never sees the rewrite — it just receives compressed output.

```bash
rtk init -g                      # Claude Code / Copilot (global, recommended)
rtk init -g --gemini             # Gemini CLI
rtk init -g --codex              # Codex / OpenAI
rtk init -g --agent cursor       # Cursor
rtk init --agent windsurf        # Windsurf (project-scoped)
rtk init --agent cline           # Cline / Roo Code (project-scoped)
rtk init -g --opencode           # OpenCode plugin
rtk init -g --auto-patch         # Non-interactive (CI/CD)
rtk init --show                  # Verify current installation
```

**After installing, restart your AI tool.**

> **Scope**: The hook only applies to Bash tool calls. Claude Code built-in tools (`Read`, `Grep`, `Glob`) bypass it. Use shell equivalents (`cat`, `rg`, `find`) or explicit `rtk` commands for those workflows.

---

## Core Commands

### Files
```bash
rtk ls .                         # Token-optimized directory tree
rtk read src/main.rs             # Smart file reading
rtk read src/main.rs -l aggressive  # Signatures only (strips function bodies)
rtk smart src/main.rs            # 2-line heuristic code summary
rtk find "*.rs" .                # Compact find results
rtk grep "fn parse" src/         # Grouped search results
rtk diff file1.rs file2.rs       # Condensed diff
```

### Git
```bash
rtk git status                   # Compact status (~200 tokens vs ~2000)
rtk git log -n 10                # One-line commits
rtk git diff                     # Condensed diff
rtk git add .                    # Output: "ok"
rtk git commit -m "fix: parse"   # Output: "ok abc1234"
rtk git push                     # Output: "ok main"
rtk git pull                     # Output: "ok 3 files +10 -2"
```

### Test Runners
```bash
rtk cargo test                   # Rust — failures only (-90%)
rtk test cargo test              # Explicit test wrapper
rtk pytest                       # Python — failures only (-90%)
rtk go test                      # Go — NDJSON compact (-90%)
rtk rspec                        # Ruby — JSON compact (-60%+)
rtk vitest run                   # Vitest — failures only
rtk playwright test              # E2E — failures only
rtk rake test                    # Ruby minitest (-90%)
```

### Build & Lint
```bash
rtk cargo build                  # Rust build (-80%)
rtk cargo clippy                 # Clippy warnings grouped (-80%)
rtk tsc                          # TypeScript errors by file
rtk lint                         # ESLint grouped by rule/file
rtk lint biome                   # Biome linter
rtk ruff check                   # Python lint (JSON, -80%)
rtk golangci-lint run            # Go lint (JSON, -85%)
rtk rubocop                      # Ruby lint (JSON, -60%+)
rtk next build                   # Next.js compact build output
rtk prettier --check .           # Files needing formatting only
```

### GitHub CLI
```bash
rtk gh pr list                   # Compact PR listing
rtk gh pr view 42                # PR details + check status
rtk gh issue list                # Compact issue listing
rtk gh run list                  # Workflow run status
```

### Containers & Infra
```bash
rtk docker ps                    # Compact container list
rtk docker images                # Compact image list
rtk docker logs <container>      # Deduplicated log lines
rtk docker compose ps            # Compose services
rtk kubectl pods                 # Compact pod list
rtk kubectl logs <pod>           # Deduplicated pod logs
rtk kubectl services             # Compact service list
```

### Data & Utilities
```bash
rtk json config.json             # Structure without values (schema view)
rtk env -f AWS                   # Filtered env vars (e.g., AWS_*)
rtk log app.log                  # Deduplicated application logs
rtk curl https://api.example.com # Auto-detect JSON + print schema
rtk deps                         # Dependency summary
rtk summary <long command>       # Heuristic summary of any command
rtk proxy <command>              # Raw passthrough + token tracking
```

### Package Managers
```bash
rtk pnpm list                    # Compact dependency tree
rtk pip list                     # Python packages (auto-detects uv)
rtk pip outdated                 # Outdated packages only
rtk bundle install               # Ruby gems (strips "Using" lines)
rtk prisma generate              # Schema generation (no ASCII art)
```

---

## Global Flags

```bash
rtk -u <command>                 # --ultra-compact: ASCII icons, inline format
rtk -v <command>                 # --verbose (stack: -v, -vv, -vvv)
```

---

## Token Savings Analytics

```bash
rtk gain                         # Summary stats
rtk gain --graph                 # ASCII graph (last 30 days)
rtk gain --history               # Recent command history with savings
rtk gain --daily                 # Day-by-day breakdown
rtk gain --all --format json     # JSON export for dashboards

rtk discover                     # Find missed savings opportunities
rtk discover --all --since 7     # All projects, last 7 days

rtk session                      # RTK adoption across recent sessions
```

---

## Real-World Usage Patterns

### Pattern 1: Rust project CI loop
```bash
# Instead of: cargo test 2>&1 (200+ lines on failure)
rtk cargo test
# Output (~20 lines):
# FAILED: 2/15 tests
#   test_edge_case: assertion failed at utils.rs:42
#   test_overflow: panic at utils.rs:18

rtk cargo clippy
# Output: grouped warnings by lint rule, not raw rustc noise
```

### Pattern 2: Python project
```bash
rtk pytest                       # Only failing tests + tracebacks
rtk ruff check                   # JSON-parsed, grouped by rule
rtk pip outdated                 # Table of outdated packages only
```

### Pattern 3: Node/TypeScript project
```bash
rtk tsc                          # TypeScript errors grouped by file
rtk lint                         # ESLint violations grouped by rule
rtk next build                   # Next.js build, key metrics only
rtk vitest run                   # Failures only, not 300 "✓ pass" lines
```

### Pattern 4: Reading large files intelligently
```bash
# Full file (smart truncation)
rtk read src/lib.rs

# Signatures only — ideal for understanding module structure
rtk read src/lib.rs -l aggressive

# 2-line summary heuristic
rtk smart src/lib.rs
```

### Pattern 5: Git workflow in AI sessions
```bash
rtk git status                   # ~200 tokens (vs raw ~2000)
rtk git diff HEAD~1              # Condensed patch
rtk git log -n 5                 # 5 one-liners
rtk git add . && rtk git commit -m "feat: add parser"
# Output: "ok" then "ok a3f9c12"
```

---

## Configuration

RTK is configuration-free by default. All tuning is via CLI flags:

| Flag | Effect |
|------|--------|
| `-u` / `--ultra-compact` | Maximum compression with ASCII icons |
| `-v` / `-vv` / `-vvv` | Increase verbosity for debugging |
| `-l aggressive` | Aggressive level for `rtk read` (signatures only) |

The `rtk init` command writes hooks to your AI tool's config directory (global `-g`) or the current project directory (no `-g`).

---

## Troubleshooting

### `rtk gain` fails after `cargo install rtk`
Wrong package from crates.io. Reinstall from source:
```bash
cargo install --git https://github.com/rtk-ai/rtk
```

### Hook not intercepting commands
```bash
rtk init --show                  # Check hook installation status
rtk init -g                      # Reinstall hook
# Then restart your AI tool (Claude Code, Cursor, etc.)
```

### Commands not compressed / hook bypassed
Built-in AI tool operations (`Read`, `Grep`, `Glob` in Claude Code) bypass the Bash hook. Use shell equivalents:
```bash
# Instead of Claude's Read tool, ask it to run:
rtk read src/main.rs
# Instead of Grep tool:
rtk grep "pattern" src/
```

### PATH not found after quick install
```bash
export PATH="$HOME/.local/bin:$PATH"
# Persist it:
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Wrong binary architecture (macOS)
Download the correct release asset:
- Apple Silicon: `rtk-aarch64-apple-darwin.tar.gz`
- Intel Mac: `rtk-x86_64-apple-darwin.tar.gz`

### Verify savings are being tracked
```bash
rtk gain --history               # Should show recent commands
rtk session                      # Should show adoption %
```
If empty, the hook may not be active. Run `rtk init --show`.

---

## How RTK Works Internally

RTK applies four compression strategies per command type:

1. **Smart Filtering** — removes comments, blank lines, boilerplate (`Using gem...`, progress bars)
2. **Grouping** — aggregates similar items (files by directory, errors by lint rule)
3. **Truncation** — keeps relevant context, discards redundant repetition
4. **Deduplication** — collapses repeated log lines with occurrence counts

```
Without RTK:
  Agent → "git status" → shell → git → ~2,000 tokens raw → Agent

With RTK:
  Agent → "git status" → RTK → git → filter/group/truncate → ~200 tokens → Agent
```

The binary runs in-process before output is returned to the shell, adding <10ms latency.

---

## Supported AI Tools Summary

| Tool | Command | Scope |
|------|---------|-------|
| Claude Code | `rtk init -g` | Global |
| GitHub Copilot | `rtk init -g` | Global |
| Cursor | `rtk init -g --agent cursor` | Global |
| Gemini CLI | `rtk init -g --gemini` | Global |
| Codex | `rtk init -g --codex` | Global |
| Windsurf | `rtk init --agent windsurf` | Project |
| Cline / Roo Code | `rtk init --agent cline` | Project |
| OpenCode | `rtk init -g --opencode` | Global |

---

## Quick Reference Card

```bash
# Setup
brew install rtk && rtk init -g && <restart AI tool>

# Most used
rtk git status / diff / log / add / commit / push
rtk cargo test / build / clippy
rtk pytest / go test / vitest run
rtk read <file> / rtk ls . / rtk grep <pat> <dir>
rtk tsc / rtk lint / rtk ruff check

# Analytics
rtk gain --graph
rtk discover

# Flags
rtk -u <cmd>   # ultra-compact
rtk -v <cmd>   # verbose/debug
```
```
