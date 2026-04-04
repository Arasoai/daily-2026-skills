```markdown
---
name: autoagent-harness-engineering
description: Autonomous agent harness engineering using AutoAgent — meta-agent that iteratively builds, benchmarks, and improves AI agent harnesses overnight
triggers:
  - set up autoagent for autonomous agent engineering
  - run the meta-agent harness loop
  - configure agent.py harness for benchmarking
  - add benchmark tasks to autoagent
  - iterate on agent harness with autoagent
  - run harbor benchmark with autoagent
  - how do I use autoagent to improve my agent
  - set up autonomous harness engineering experiment
---

# AutoAgent Harness Engineering

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

AutoAgent is an autonomous harness engineering framework. You give an AI agent a task, and it builds and iterates on an agent harness overnight — modifying system prompts, tools, agent configuration, and orchestration, then running benchmarks, checking scores, and keeping or discarding changes automatically. Think of it as autoresearch, but for agent engineering.

---

## Installation & Setup

### Requirements

- Docker (Desktop or Engine)
- Python 3.10+
- [uv](https://docs.astral.sh/uv/)
- Model provider credentials (e.g., `OPENAI_API_KEY`)

### Install

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and install dependencies
git clone https://github.com/kevinrgu/autoagent
cd autoagent
uv sync

# Configure environment variables
cat > .env << 'EOF'
OPENAI_API_KEY=$OPENAI_API_KEY
# Add other provider keys as needed
EOF

# Build the base Docker image
docker build -f Dockerfile.base -t autoagent-base .
```

---

## Project Structure

```
autoagent/
├── agent.py          # Single-file harness under test (primary edit surface)
├── program.md        # Meta-agent instructions + directive (human edits this)
├── Dockerfile.base   # Base Docker image for task containers
├── tasks/            # Benchmark tasks in Harbor format
│   └── my-task/
│       ├── task.toml
│       ├── instruction.md
│       ├── tests/
│       │   ├── test.sh
│       │   └── test.py
│       ├── environment/
│       │   └── Dockerfile
│       └── files/
├── jobs/             # Harbor job outputs (auto-generated)
├── results.tsv       # Experiment log (created by meta-agent, gitignored)
├── run.log           # Latest run output
└── .agent/           # Optional workspace artifacts
```

---

## Key Concepts

### The Two Files You Actually Edit

1. **`program.md`** — Instructions for the meta-agent + the directive (what kind of agent to build). **You edit this.**
2. **`agent.py`** — The entire harness under test. Contains config, tool definitions, agent registry, routing/orchestration, and the Harbor adapter (fixed section). **The meta-agent edits this.**

### The Loop

```
meta-agent reads program.md
    → inspects current agent.py
    → runs benchmark
    → diagnoses failures
    → modifies agent.py (prompt, tools, config, orchestration)
    → runs benchmark again
    → keeps change if score improves, discards if not
    → repeats
```

---

## Running Benchmarks

### Run a Single Task

```bash
rm -rf jobs
mkdir -p jobs
uv run harbor run \
  -p tasks/ \
  --task-name "<task-name>" \
  -l 1 \
  -n 1 \
  --agent-import-path agent:AutoAgent \
  -o jobs \
  --job-name latest > run.log 2>&1
```

### Run All Tasks in Parallel

```bash
rm -rf jobs
mkdir -p jobs
uv run harbor run \
  -p tasks/ \
  -n 100 \
  --agent-import-path agent:AutoAgent \
  -o jobs \
  --job-name latest > run.log 2>&1
```

**Flags:**
| Flag | Description |
|---|---|
| `-p tasks/` | Path to tasks directory |
| `--task-name` | Run a specific named task |
| `-l 1` | Limit to 1 task |
| `-n <N>` | Concurrency (default 4, use 100 for max parallelism) |
| `--agent-import-path` | Python import path to your agent class |
| `-o jobs` | Output directory for job results |
| `--job-name` | Label for this run |

### Launching the Meta-Agent

Point any coding agent (Claude Code, Cursor, Codex, etc.) at the repo and say:

```
Read program.md and let's kick off a new experiment!
```

---

## Task Format (Harbor)

Tasks live in `tasks/` following the [Harbor task format](https://harborframework.com/docs/tasks).

### Minimal Task Structure

```
tasks/my-task/
├── task.toml           # Config (timeouts, metadata)
├── instruction.md      # Prompt sent to the agent
├── tests/
│   ├── test.sh         # Entry point — writes /logs/reward.txt
│   └── test.py         # Verification logic
├── environment/
│   └── Dockerfile      # FROM autoagent-base
└── files/              # Reference files mounted into container
```

### `task.toml` Example

```toml
[task]
name = "my-task"
description = "A sample benchmark task"
timeout = 300

[task.metadata]
category = "reasoning"
difficulty = "medium"
```

### `instruction.md` Example

```markdown
You are given a dataset of customer reviews. Your task is to:

1. Classify each review as positive, negative, or neutral
2. Extract the main topic of each review
3. Output a JSON file at /output/results.json

The reviews are located at /files/reviews.csv
```

### `tests/test.sh` Example

```bash
#!/bin/bash
set -e

# Run verification and write score to /logs/reward.txt
python /tests/test.py
```

### `tests/test.py` Example

```python
import json
import os

EXPECTED_COUNT = 50
reward_path = "/logs/reward.txt"
output_path = "/output/results.json"

try:
    with open(output_path) as f:
        results = json.load(f)

    # Score based on completeness and format
    score = 0.0

    if isinstance(results, list):
        score += 0.3  # correct format

    correct = sum(1 for r in results if "label" in r and "topic" in r)
    score += 0.7 * (correct / EXPECTED_COUNT)

    score = min(1.0, max(0.0, score))

except Exception as e:
    score = 0.0

os.makedirs("/logs", exist_ok=True)
with open(reward_path, "w") as f:
    f.write(str(score))
```

### `environment/Dockerfile` Example

```dockerfile
FROM autoagent-base

# Add any task-specific dependencies
RUN pip install pandas scikit-learn

# Copy reference files
COPY files/ /files/
```

---

## `agent.py` Structure

`agent.py` is the single-file harness. It has two sections:

### Editable Harness Section (meta-agent modifies this)

```python
# ── CONFIG ──────────────────────────────────────────────────────────────────
MODEL = "gpt-4o"
TEMPERATURE = 0.0
MAX_TOKENS = 4096
SYSTEM_PROMPT = """You are a helpful AI assistant..."""

# ── TOOL DEFINITIONS ─────────────────────────────────────────────────────────
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read a file from the filesystem",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "File path to read"}
                },
                "required": ["path"]
            }
        }
    },
    # ... more tools
]

# ── AGENT REGISTRY ───────────────────────────────────────────────────────────
AGENTS = {
    "default": {
        "model": MODEL,
        "system_prompt": SYSTEM_PROMPT,
        "tools": TOOLS,
        "temperature": TEMPERATURE,
    }
}

# ── ROUTING / ORCHESTRATION ──────────────────────────────────────────────────
def route(task: dict) -> str:
    """Return agent name based on task properties."""
    return "default"
```

### Fixed Adapter Section (do NOT modify)

```python
# ── HARBOR ADAPTER (FIXED — DO NOT EDIT) ─────────────────────────────────────
class AutoAgent:
    """Harbor-compatible agent entry point."""
    # ... Harbor integration + trajectory serialization
```

---

## `program.md` Template

```markdown
# Meta-Agent Program

## Context
This repo implements an autonomous harness engineering loop. The meta-agent
reads this file, inspects agent.py, runs benchmarks, modifies the harness,
and iterates to maximize benchmark score.

## Directive
Build an agent that can [DESCRIBE YOUR TASK HERE].

## Constraints
- Focus on [specific capabilities]
- The agent should prioritize [accuracy/speed/cost]
- Target benchmark: [task name]

## Current Status
- Baseline score: [X.X]
- Last experiment: [description]
- Next hypothesis: [what to try]

## Experiment Log
| Run | Change | Score | Delta |
|-----|--------|-------|-------|
| 1   | baseline | 0.42 | - |
```

---

## Common Patterns

### Pattern 1: Multi-Agent Routing

```python
# In agent.py editable section
AGENTS = {
    "classifier": {
        "model": "gpt-4o-mini",
        "system_prompt": "You classify tasks by type. Output JSON.",
        "tools": [],
        "temperature": 0.0,
    },
    "executor": {
        "model": "gpt-4o",
        "system_prompt": "You execute complex tasks with tools.",
        "tools": TOOLS,
        "temperature": 0.1,
    },
    "verifier": {
        "model": "gpt-4o",
        "system_prompt": "You verify outputs for correctness.",
        "tools": [],
        "temperature": 0.0,
    }
}

def route(task: dict) -> str:
    instruction = task.get("instruction", "").lower()
    if "verify" in instruction or "check" in instruction:
        return "verifier"
    elif len(instruction) < 100:
        return "classifier"
    return "executor"
```

### Pattern 2: Tool-Heavy Harness

```python
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "bash",
            "description": "Run a bash command and return output",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {"type": "string"}
                },
                "required": ["command"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Write content to a file",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string"},
                    "content": {"type": "string"}
                },
                "required": ["path", "content"]
            }
        }
    },
]
```

### Pattern 3: Chain-of-Thought System Prompt

```python
SYSTEM_PROMPT = """You are an expert AI assistant.

Before answering, always:
1. Restate the task in your own words
2. Break it into subtasks
3. Execute each subtask step by step
4. Verify your output

Use tools liberally. If a tool fails, retry with different parameters.
Always write final outputs to the specified output path.
"""
```

---

## Cleanup

Docker images and containers accumulate across runs:

```bash
# Clean Harbor's cached task images and task cache
uv run harbor cache clean -f

# Full Docker cleanup (all unused images, build cache)
docker system prune -a -f

# Lighter: just dead containers
docker container prune -f

# If Docker becomes unresponsive after many concurrent runs
killall Docker && open -a Docker
```

---

## Troubleshooting

### Score is always 0.0

- Check `/logs/reward.txt` exists and contains a float between 0.0 and 1.0
- Inspect `jobs/latest/` for task-specific logs
- Run `cat run.log` to see Harbor output and errors
- Verify `test.sh` is executable: `chmod +x tasks/my-task/tests/test.sh`

### Docker build fails

```bash
# Ensure base image exists
docker images | grep autoagent-base

# Rebuild if missing
docker build -f Dockerfile.base -t autoagent-base .
```

### Agent import error

```bash
# Verify agent.py exports AutoAgent
python -c "from agent import AutoAgent; print('OK')"

# Check import path flag matches
--agent-import-path agent:AutoAgent
```

### Out of memory / slow runs

```bash
# Reduce concurrency
uv run harbor run -p tasks/ -n 4 --agent-import-path agent:AutoAgent -o jobs --job-name latest

# Clean up between runs
rm -rf jobs && mkdir -p jobs
```

### Meta-agent not improving scores

- Update `program.md` with clearer directives and constraints
- Check `results.tsv` for experiment history
- Add `.agent/` workspace files with domain-specific context
- Ensure `tasks/` contains representative, well-scored tasks

---

## Environment Variables Reference

```bash
# OpenAI
OPENAI_API_KEY=...

# Anthropic (if using Claude)
ANTHROPIC_API_KEY=...

# Google (if using Gemini)
GOOGLE_API_KEY=...

# Azure OpenAI (if applicable)
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_ENDPOINT=...
```

Set these in `.env` at the project root — `uv run` loads them automatically via python-dotenv or your shell.

---

## Quick Reference

| Action | Command |
|---|---|
| Install deps | `uv sync` |
| Build base image | `docker build -f Dockerfile.base -t autoagent-base .` |
| Run single task | `uv run harbor run -p tasks/ --task-name <name> -n 1 --agent-import-path agent:AutoAgent -o jobs --job-name latest` |
| Run all tasks | `uv run harbor run -p tasks/ -n 100 --agent-import-path agent:AutoAgent -o jobs --job-name latest` |
| View run logs | `cat run.log` |
| View job outputs | `ls jobs/latest/` |
| Clean Harbor cache | `uv run harbor cache clean -f` |
| Clean Docker | `docker system prune -a -f` |
| Start meta-agent | Say: *"Read program.md and let's kick off a new experiment!"* |
```
