```markdown
---
name: clawteam-agent-swarm
description: Expert skill for using ClawTeam to orchestrate AI agent swarms with one command for full automation of complex tasks
triggers:
  - set up ClawTeam agent swarm
  - spawn multiple AI agents to work together
  - orchestrate agents with ClawTeam
  - automate tasks with agent team
  - use ClawTeam for multi-agent workflow
  - coordinate Claude Code agents in parallel
  - run distributed AI research with ClawTeam
  - build swarm intelligence pipeline
---

# ClawTeam Agent Swarm Intelligence

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ClawTeam enables AI agents to self-organize into collaborative swarms. One command launches a leader agent that spawns specialized sub-agents, each with isolated git worktrees and tmux windows, coordinating via CLI to complete complex tasks in parallel — zero human orchestration required.

---

## Installation

```bash
pip install clawteam
```

Requires Python ≥ 3.10 and `tmux` installed on your system.

```bash
# Verify installation
clawteam --version

# Optional: install tmux if missing
# macOS
brew install tmux
# Ubuntu/Debian
sudo apt install tmux
```

---

## Core Concepts

| Concept | What It Is |
|---|---|
| **Team** | Named group of agents sharing a workspace |
| **Leader agent** | Orchestrates the swarm; spawns/monitors workers |
| **Worker agent** | Runs in isolated git worktree + tmux window |
| **Inbox** | File-based or ZeroMQ message passing between agents |
| **Board** | Monitoring dashboard (CLI or web) |
| **Task** | Unit of work with owner, status, and optional blockers |

---

## Quick Start (3 Minutes)

### 1. Create a team

```bash
clawteam team create my-team --description "My first swarm"
```

### 2. Spawn the leader agent (Claude Code example)

```bash
clawteam spawn \
  --team my-team \
  --agent-name leader \
  --agent-cmd "claude" \
  --task "You are the team leader. Build a REST API with auth and tests. Use clawteam to spawn workers."
```

### 3. Watch the swarm

```bash
# Tiled tmux view of all agents
clawteam board attach my-team

# Web dashboard
clawteam board serve --port 8080
```

---

## Key CLI Commands

### Team Management

```bash
# Create a team
clawteam team create <team-name> --description "..."

# List teams
clawteam team list

# Remove a team and clean up worktrees
clawteam team destroy <team-name>

# Spawn an entire team from a TOML template
clawteam team spawn-team <team-name> --template templates/engineering.toml
```

### Spawning Agents

```bash
# Spawn a Claude Code worker
clawteam spawn \
  --team my-team \
  --agent-name worker1 \
  --task "Implement the authentication module"

# Spawn a Codex worker
clawteam spawn \
  --team my-team \
  --agent-name worker2 \
  --agent-cmd "codex" \
  --task "Write PostgreSQL schema and migrations"

# Spawn with a custom CLI agent
clawteam spawn \
  --team my-team \
  --agent-name worker3 \
  --agent-cmd "my-custom-agent" \
  --task "Build React frontend components"

# Kill a specific agent
clawteam kill my-team worker1
```

### Task Management

```bash
# Create a task
clawteam task create my-team \
  --title "Build auth module" \
  --owner worker1

# Create a task with dependencies
clawteam task create my-team \
  --title "Integration tests" \
  --owner tester \
  --blocked-by T1 --blocked-by T2

# List tasks (filter by owner)
clawteam task list my-team
clawteam task list my-team --owner worker1

# Update task status
clawteam task update my-team T1 --status done

# Show task board
clawteam board show my-team
```

### Inter-Agent Messaging (Inbox)

```bash
# Send a message to another agent
clawteam inbox send my-team leader "Auth complete. All 42 tests passing."
clawteam inbox send my-team worker2 "Schema approved. Start migrations."

# Read your inbox
clawteam inbox read my-team --agent me

# List all messages in team inbox
clawteam inbox list my-team
```

### Monitoring

```bash
# Show board summary
clawteam board show my-team

# Attach to tiled tmux view
clawteam board attach my-team

# Start web UI
clawteam board serve --port 8080

# Show agent logs
clawteam logs my-team worker1
```

---

## TOML Team Templates

Define reusable swarm configurations in TOML:

```toml
# templates/fullstack.toml
[team]
name = "webapp"
description = "Full-stack web application team"

[[agents]]
name = "architect"
task = "Design REST API schema and write OpenAPI spec. Save to docs/api.yaml."

[[agents]]
name = "backend1"
task = "Implement JWT authentication using docs/api.yaml as spec."
blocked_by = ["architect"]

[[agents]]
name = "backend2"
task = "Build PostgreSQL models and migrations using docs/api.yaml as spec."
blocked_by = ["architect"]

[[agents]]
name = "frontend"
task = "Build React frontend consuming the API defined in docs/api.yaml."
blocked_by = ["architect"]

[[agents]]
name = "tester"
task = "Write pytest integration tests covering auth, CRUD, and edge cases."
blocked_by = ["backend1", "backend2", "frontend"]
```

```bash
clawteam team spawn-team webapp --template templates/fullstack.toml
```

---

## Real-World Examples

### Example 1: Autonomous ML Research (8 GPUs)

```bash
# Human gives one prompt to the leader agent:
# "Use 8 GPUs to optimize train.py. Read program.md for instructions."

# The leader agent runs these commands autonomously:
clawteam team create autoresearch --description "LLM training optimization"

# Spawn one worker per GPU with a research direction
clawteam spawn --team autoresearch --agent-name gpu0 \
  --task "Explore model depth: vary DEPTH from 10 to 16, record val_bpb to results.tsv"

clawteam spawn --team autoresearch --agent-name gpu1 \
  --task "Explore model width: vary ASPECT_RATIO from 80 to 128, record val_bpb to results.tsv"

clawteam spawn --team autoresearch --agent-name gpu2 \
  --task "Tune learning rates and optimizer: try AdamW schedules, record to results.tsv"

clawteam spawn --team autoresearch --agent-name gpu3 \
  --task "Explore batch sizes from 2^14 to 2^18 with gradient accumulation, record to results.tsv"

# Leader monitors and cross-pollinates every 30 minutes
clawteam board show autoresearch

# After results are in, leader sends best config to new agents
clawteam inbox send autoresearch gpu4 \
  "Best config so far: depth=12, batch=2^17, norm-before-RoPE. Start from this baseline."
```

### Example 2: Full-Stack App in Python

```python
# Use ClawTeam programmatically via subprocess in a Python orchestration script
import subprocess
import json

def spawn_agent(team: str, name: str, task: str, agent_cmd: str = "claude") -> None:
    subprocess.run([
        "clawteam", "spawn",
        "--team", team,
        "--agent-name", name,
        "--agent-cmd", agent_cmd,
        "--task", task,
    ], check=True)

def send_message(team: str, to: str, message: str) -> None:
    subprocess.run([
        "clawteam", "inbox", "send", team, to, message
    ], check=True)

def get_board(team: str) -> str:
    result = subprocess.run(
        ["clawteam", "board", "show", team, "--json"],
        capture_output=True, text=True, check=True
    )
    return json.loads(result.stdout)

# Orchestrate a data pipeline team
team = "data-pipeline"
subprocess.run(["clawteam", "team", "create", team], check=True)

spawn_agent(team, "ingester", "Build a CSV ingestion module that reads from ./data/raw/")
spawn_agent(team, "transformer", "Build a pandas transformation pipeline for the ingested data")
spawn_agent(team, "loader", "Build a SQLite loader that writes transformed data to ./data/output.db")
spawn_agent(team, "tester", "Write pytest tests for all three pipeline stages")

# Monitor until done
board = get_board(team)
print(json.dumps(board, indent=2))
```

### Example 3: AI Hedge Fund Swarm

```bash
# Spawn a financial analysis team
clawteam team create hedgefund --description "Market analysis swarm"

clawteam spawn --team hedgefund --agent-name researcher \
  --task "Scrape and analyze AAPL, MSFT, NVDA earnings data for Q1 2026. Save to reports/fundamentals.json"

clawteam spawn --team hedgefund --agent-name quant \
  --task "Build momentum and mean-reversion signals from reports/fundamentals.json"

clawteam spawn --team hedgefund --agent-name risk \
  --task "Run VaR and stress tests on the portfolio from quant's output. Save to reports/risk.json"

clawteam spawn --team hedgefund --agent-name trader \
  --task "Generate trade recommendations from risk report. Output to reports/trades.json"

# Watch all agents in split view
clawteam board attach hedgefund
```

### Example 4: Writing a Custom Leader Prompt

When talking to Claude Code or Codex as the leader, include ClawTeam context:

```text
You are the team leader for a software project. You have access to ClawTeam CLI commands.

Your workflow:
1. Break the task into parallel subtasks
2. Run: clawteam team create <name>
3. Spawn workers: clawteam spawn --team <name> --agent-name <name> --task "..."
4. Monitor: clawteam board show <name>
5. Coordinate via: clawteam inbox send <team> <agent> "<message>"
6. When a worker finishes, run: clawteam task update <team> <id> --status done

Task: Build a microservices API with user service, product service, and API gateway.
Use git worktrees so each service is isolated. Spawn one agent per service.
```

---

## Transport Backends

ClawTeam supports two transport modes:

```bash
# File-based (default) — no dependencies, works everywhere
clawteam team create my-team --transport file

# ZeroMQ P2P — low latency, better for large swarms
clawteam team create my-team --transport zmq --zmq-port 5555
```

---

## Git Worktree Isolation

Each spawned agent automatically gets:
- A dedicated `git worktree` (real branch, real diffs)
- Its own tmux window
- Isolated working directory under `.clawteam/worktrees/<agent-name>/`

```bash
# View all worktrees created by ClawTeam
git worktree list

# Merge a worker's branch after review
git merge clawteam/worker1

# Clean up all worktrees for a team
clawteam team destroy my-team  # removes worktrees automatically
```

---

## Configuration

ClawTeam reads from `~/.clawteam/config.toml` or a local `.clawteam.toml`:

```toml
[defaults]
agent_cmd = "claude"          # default agent CLI command
transport = "file"            # "file" or "zmq"
worktree_base = ".clawteam/worktrees"

[board]
web_port = 8080
refresh_interval = 5          # seconds

[zmq]
base_port = 5555
```

---

## Troubleshooting

### tmux session not found
```bash
# List active tmux sessions
tmux ls

# ClawTeam creates sessions named clawteam-<team>
tmux attach -t clawteam-my-team
```

### Agent not receiving tasks
```bash
# Check inbox directly
clawteam inbox list my-team

# Verify agent is running
clawteam board show my-team

# Re-send message
clawteam inbox send my-team worker1 "Your task: implement auth. Check tasks with: clawteam task list my-team"
```

### Worktree conflicts
```bash
# List and prune stale worktrees
git worktree list
git worktree prune

# Force remove a specific worktree
git worktree remove .clawteam/worktrees/worker1 --force
```

### Clean up a stuck team
```bash
# Kill all agents in a team
clawteam team destroy my-team --force

# Or kill tmux session directly
tmux kill-session -t clawteam-my-team
```

### ZeroMQ connection refused
```bash
# Check if port is in use
lsof -i :5555

# Use a different port
clawteam team create my-team --transport zmq --zmq-port 5556
```

---

## Compatible Agents

| Agent | Command Flag | Notes |
|---|---|---|
| Claude Code | `--agent-cmd claude` | Default, best orchestration |
| OpenAI Codex | `--agent-cmd codex` | Good for parallel coding tasks |
| OpenClaw | `--agent-cmd openclaw` | Open-source Claude Code alternative |
| nanobot | `--agent-cmd nanobot` | Lightweight option |
| Cursor | Via tmux integration | IDE-based |
| Custom | `--agent-cmd ./my-agent.sh` | Any CLI that reads stdin |

---

## Project Links

- **GitHub**: https://github.com/HKUDS/ClawTeam
- **AutoResearch results**: https://github.com/novix-science/autoresearch
- **OpenClaw**: https://github.com/nicepkg/OpenClaw
- **nanobot**: https://github.com/AbanteAI/nanobot
```
