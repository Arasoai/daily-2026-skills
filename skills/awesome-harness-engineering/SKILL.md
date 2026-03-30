```markdown
---
name: awesome-harness-engineering
description: Curated knowledge base for harness engineering — the practice of shaping the environment around AI agents for reliability, context management, evaluation, and safe autonomy.
triggers:
  - harness engineering for AI agents
  - how to build an agent harness
  - context engineering for coding agents
  - agent evaluation and observability
  - safe autonomy and guardrails for agents
  - AGENTS.md or CLAUDE.md setup
  - benchmarking agent harness quality
  - long-running agent workflow design
---

# Awesome Harness Engineering

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What Is Harness Engineering?

Harness engineering is the discipline of shaping the **environment around an AI agent** so it can work reliably. The harness is everything except the model itself:

- **Context & memory management** — what the agent sees and remembers
- **Constraints & guardrails** — what the agent is allowed to do
- **Specs & agent files** — repo-local instructions that persist across sessions
- **Evals & observability** — how you measure and debug agent behavior
- **Orchestration & runtime** — how tasks are scheduled, retried, and handed off

The key insight: *weak results from coding agents are usually harness problems, not model problems.*

---

## Core Mental Model

```
┌─────────────────────────────────────────┐
│              AGENT HARNESS              │
│                                         │
│  ┌─────────┐   ┌──────────────────┐    │
│  │ Prompts │   │  Context Window  │    │
│  │ & Specs │   │  (working memory)│    │
│  └─────────┘   └──────────────────┘    │
│                                         │
│  ┌──────────┐  ┌──────────────────┐    │
│  │  Tools   │  │  Guardrails &    │    │
│  │ (bounded)│  │  Sandboxing      │    │
│  └──────────┘  └──────────────────┘    │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │  Evals, Traces & Observability   │  │
│  └──────────────────────────────────┘  │
│                                         │
│              ┌───────┐                 │
│              │ MODEL │                 │
│              └───────┘                 │
└─────────────────────────────────────────┘
```

---

## 1. Repo-Local Agent Instructions

### AGENTS.md / CLAUDE.md / agent.md

Place a file in the root of your repo to give agents persistent, repo-scoped instructions.

**`AGENTS.md` (open format — works with many agents):**

```markdown
# Agent Instructions

## Repo Overview
This is a TypeScript monorepo. Packages are under `packages/`.

## Development Commands
- `pnpm install` — install dependencies
- `pnpm build` — build all packages
- `pnpm test` — run all tests
- `pnpm lint` — lint all packages

## Coding Conventions
- Use named exports only
- All async functions must handle errors explicitly
- Write tests in Vitest; place them next to the source file as `*.test.ts`

## What NOT to Do
- Do not modify `packages/core/src/generated/` — these are auto-generated
- Do not commit `.env` files
- Do not use `any` in TypeScript

## Verification Steps
After making changes, always run:
1. `pnpm build`
2. `pnpm test`
3. `pnpm lint`
```

**`CLAUDE.md` (Claude Code specific):**

```markdown
# Claude Instructions

## Project Context
REST API service using Express + Prisma + PostgreSQL.

## Environment Setup
\`\`\`bash
cp .env.example .env
# Set DATABASE_URL in .env
pnpm install
pnpm db:migrate
\`\`\`

## Key Patterns
- Route handlers go in `src/routes/`
- Business logic goes in `src/services/`
- Database queries go in `src/repositories/`
- Always validate input with Zod schemas in `src/schemas/`

## Testing
- Unit tests: `pnpm test:unit`
- Integration tests (requires DB): `pnpm test:integration`
- Check coverage: `pnpm test:coverage`

## Agent Checkpoints
After each feature, verify:
- [ ] Types compile: `pnpm typecheck`
- [ ] Tests pass: `pnpm test`
- [ ] No lint errors: `pnpm lint`
```

---

## 2. Context Engineering Patterns

### Treat Context as a Budget

```python
# Pattern: Context budget management
# Don't dump everything — be selective about what enters context

CONTEXT_BUDGET = {
    "system_prompt": 2000,      # tokens — keep it tight
    "task_spec": 3000,          # the current task description
    "relevant_files": 20000,    # only files the agent needs RIGHT NOW
    "tool_results": 10000,      # recent tool outputs (rolling window)
    "conversation": 5000,       # recent turns only
    "reserve": 10000,           # leave room for model output
}

# Rolling window for tool results — don't let noisy output fill context
def add_tool_result(context_window: list, result: str, max_tokens: int = 2000):
    """Add tool result, truncating if needed, keeping failures visible."""
    truncated = result[:max_tokens] if len(result) > max_tokens else result
    context_window.append({
        "role": "tool",
        "content": truncated,
        "truncated": len(result) > max_tokens
    })
    return context_window
```

### KV-Cache Locality (from Manus playbook)

```python
# Pattern: Keep stable content at the TOP of context to maximize cache hits
# System prompt (never changes) → repo instructions (rarely changes) → task state → recent actions

def build_context(system_prompt, repo_instructions, task_state, recent_actions):
    """
    Order matters for KV-cache efficiency.
    Stable content first = more cache hits = lower latency + cost.
    """
    return [
        {"role": "system", "content": system_prompt},          # STABLE — cache hit
        {"role": "user", "content": repo_instructions},        # STABLE — cache hit  
        {"role": "user", "content": f"Current task:\n{task_state}"},   # changes per task
        *recent_actions,                                        # changes every step
    ]
```

### Filesystem as External Memory

```python
# Pattern: Use files to offload memory across context windows
import json
from pathlib import Path
from datetime import datetime

class AgentWorkingState:
    """Persist agent state to filesystem so it survives context resets."""
    
    def __init__(self, workspace: Path):
        self.workspace = workspace
        self.state_file = workspace / ".agent_state.json"
    
    def save(self, state: dict):
        state["updated_at"] = datetime.utcnow().isoformat()
        self.state_file.write_text(json.dumps(state, indent=2))
    
    def load(self) -> dict:
        if not self.state_file.exists():
            return {"tasks": [], "completed": [], "notes": []}
        return json.loads(self.state_file.read_text())
    
    def add_note(self, note: str):
        """Agent can leave notes for itself across context windows."""
        state = self.load()
        state["notes"].append({"note": note, "at": datetime.utcnow().isoformat()})
        self.save(state)
    
    def mark_complete(self, task_id: str, artifact_path: str):
        state = self.load()
        state["completed"].append({
            "task_id": task_id,
            "artifact": artifact_path,
            "at": datetime.utcnow().isoformat()
        })
        self.save(state)
```

---

## 3. Initializer Agent Pattern (Anthropic)

For long-running tasks, use a lightweight **initializer agent** to set up the harness before the main agent runs:

```python
# Pattern: Initializer agent creates the harness for the worker agent

INITIALIZER_PROMPT = """
You are an initializer agent. Your job is to:
1. Read the feature request
2. Create a feature list file: .agent/feature_list.md
3. Create an init script: .agent/init.sh  
4. Create a verification script: .agent/verify.sh
5. Write a handoff summary: .agent/handoff.md

Do NOT implement the feature. Only set up the harness.
"""

WORKER_PROMPT_TEMPLATE = """
You are a coding agent. The initializer has prepared your harness.

Before starting, run: bash .agent/init.sh
Your tasks are in: .agent/feature_list.md

After each task:
1. Run bash .agent/verify.sh
2. If verification passes, mark the task complete in .agent/feature_list.md
3. If you're approaching context limits, write a handoff note to .agent/handoff.md

Feature list:
{feature_list}
"""
```

**`.agent/init.sh` example:**

```bash
#!/bin/bash
set -e

echo "=== Agent Environment Setup ==="

# Verify we're in the right repo
if [ ! -f "package.json" ]; then
  echo "ERROR: Not in project root" && exit 1
fi

# Install deps if needed
if [ ! -d "node_modules" ]; then
  echo "Installing dependencies..."
  npm install
fi

# Run existing tests to establish baseline
echo "Baseline test run..."
npm test -- --passWithNoTests 2>&1 | tail -5

echo "=== Setup complete. Baseline established. ==="
```

**`.agent/verify.sh` example:**

```bash
#!/bin/bash
set -e

echo "=== Verification ==="
npm run typecheck && echo "✓ Types OK"
npm test && echo "✓ Tests OK"  
npm run lint && echo "✓ Lint OK"
echo "=== All checks passed ==="
```

---

## 4. 12-Factor Agent Principles

Key principles from the 12-Factor Agents and 12-Factor AgentOps frameworks:

```python
# Factor 1: Explicit prompts — no magic, prompts are code
SYSTEM_PROMPT = Path("prompts/system.md").read_text()  # versioned, reviewable

# Factor 2: Own your state — don't rely on conversation history alone
class AgentState(BaseModel):
    task_id: str
    status: Literal["pending", "running", "paused", "complete", "failed"]
    current_step: int
    artifacts: list[str] = []
    last_checkpoint: str | None = None

# Factor 3: Clean pause/resume — agent can stop and restart
def checkpoint(state: AgentState, storage_path: Path):
    """Save state so the agent can resume from here."""
    storage_path.write_text(state.model_dump_json(indent=2))

def resume(storage_path: Path) -> AgentState:
    return AgentState.model_validate_json(storage_path.read_text())

# Factor 4: Humans in the loop at the right points
def requires_human_approval(action: str, risk_level: str) -> bool:
    """Don't gate everything — only irreversible or high-risk actions."""
    HIGH_RISK = {"delete_database", "deploy_production", "send_email_blast"}
    return action in HIGH_RISK or risk_level == "critical"
```

---

## 5. Tool Design for Agents

```python
# Pattern: Tools should be easy to call correctly and hard to misuse

from typing import Annotated
from pydantic import BaseModel, Field

class ReadFileInput(BaseModel):
    path: Annotated[str, Field(description="Relative path from repo root. Example: src/index.ts")]
    max_lines: Annotated[int, Field(default=200, le=500, description="Max lines to return. Keep small to save context.")]

class WriteFileInput(BaseModel):
    path: Annotated[str, Field(description="Relative path from repo root")]
    content: str
    create_dirs: Annotated[bool, Field(default=True, description="Create parent directories if they don't exist")]

class RunCommandInput(BaseModel):
    command: Annotated[str, Field(description="Shell command to run")]
    timeout_seconds: Annotated[int, Field(default=30, le=120)]
    working_dir: Annotated[str, Field(default=".", description="Directory to run command in")]

# Good tool: returns structured, context-efficient output
def run_tests(pattern: str = "") -> dict:
    """
    Run tests matching pattern. Returns pass/fail summary + failures only.
    Full output is NOT returned to save context tokens.
    """
    import subprocess
    result = subprocess.run(
        ["npm", "test", "--", pattern, "--reporter=json"],
        capture_output=True, text=True, timeout=60
    )
    data = json.loads(result.stdout) if result.returncode == 0 else {}
    return {
        "passed": data.get("numPassedTests", 0),
        "failed": data.get("numFailedTests", 0),
        "failures": [
            {"name": t["fullName"], "error": t["failureMessages"][0][:500]}
            for suite in data.get("testResults", [])
            for t in suite.get("testResults", [])
            if t["status"] == "failed"
        ]
    }
```

---

## 6. Evaluation Patterns

### JSONL Trace Format

```python
# Pattern: Log agent traces as JSONL for eval replay

import json
from datetime import datetime
from pathlib import Path

class AgentTracer:
    def __init__(self, trace_path: Path):
        self.trace_path = trace_path
        self.trace_path.parent.mkdir(parents=True, exist_ok=True)
    
    def log(self, event_type: str, data: dict):
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "type": event_type,  # "llm_call", "tool_call", "tool_result", "checkpoint"
            **data
        }
        with self.trace_path.open("a") as f:
            f.write(json.dumps(entry) + "\n")

# Usage
tracer = AgentTracer(Path("traces/run_001.jsonl"))
tracer.log("llm_call", {"model": "claude-opus-4", "input_tokens": 4200, "output_tokens": 312})
tracer.log("tool_call", {"tool": "run_tests", "args": {"pattern": "auth"}})
tracer.log("tool_result", {"tool": "run_tests", "result": {"passed": 12, "failed": 1}})
```

### Deterministic Eval Checks

```python
# Pattern: Turn traces into repeatable evals

def eval_coding_task(trace_path: Path, expected: dict) -> dict:
    """Grade an agent trace against expected outcomes."""
    events = [json.loads(l) for l in trace_path.read_text().splitlines()]
    
    results = {}
    
    # Did the agent run verification before finishing?
    tool_calls = [e for e in events if e["type"] == "tool_call"]
    results["ran_verification"] = any(
        e["tool"] in ("run_tests", "run_lint", "run_typecheck") 
        for e in tool_calls
    )
    
    # Did tests pass at the end?
    final_test = next(
        (e for e in reversed(events) 
         if e["type"] == "tool_result" and e.get("tool") == "run_tests"),
        None
    )
    results["tests_pass"] = (
        final_test is not None and final_test["result"]["failed"] == 0
    )
    
    # Context efficiency: did it stay under budget?
    llm_calls = [e for e in events if e["type"] == "llm_call"]
    total_tokens = sum(e.get("input_tokens", 0) for e in llm_calls)
    results["under_token_budget"] = total_tokens < expected.get("max_tokens", 100000)
    
    results["score"] = sum(results.values()) / len(results)
    return results
```

---

## 7. Spec-Driven Development

```markdown
# Pattern: Write specs before handing off to agents
# File: specs/auth-feature.md

## Feature: JWT Authentication

### Acceptance Criteria
- [ ] POST /auth/login accepts {email, password}, returns {token, refreshToken}
- [ ] Token expires in 15 minutes
- [ ] Refresh token expires in 7 days  
- [ ] Invalid credentials return 401 with message "Invalid credentials"
- [ ] Rate limit: max 5 failed attempts per IP per 15 minutes

### Technical Constraints
- Use existing User model in prisma/schema.prisma
- JWT secret from process.env.JWT_SECRET
- Use bcrypt for password comparison (already installed)
- Place route in src/routes/auth.ts
- Place service in src/services/auth.ts

### Verification
\`\`\`bash
npm test -- auth
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"wrong"}' \
  -w "\nStatus: %{http_code}\n"
# Expected: Status 401
\`\`\`
```

---

## 8. Infrastructure Noise Awareness

Based on Anthropic's research, **runtime configuration can shift benchmark scores by more than many model-vs-model gaps**. Key sources of noise to control:

```python
# Harness variables to lock down for reproducible evals

EVAL_HARNESS_CONFIG = {
    # Model settings
    "model": "claude-opus-4",
    "temperature": 0.0,           # deterministic
    "max_tokens": 4096,
    
    # Environment
    "timeout_per_step_seconds": 30,
    "max_steps": 50,
    "working_dir": "/tmp/eval_workspace",  # clean slate
    
    # Tool behavior
    "shell_timeout": 30,
    "test_timeout": 60,
    
    # Context settings
    "max_context_tokens": 100000,
    "tool_output_truncation": 2000,
}

# Log the full config with every eval run so results are reproducible
def run_eval(task: dict, config: dict = EVAL_HARNESS_CONFIG) -> dict:
    run_id = f"eval_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}"
    Path(f"eval_runs/{run_id}/config.json").write_text(json.dumps(config, indent=2))
    # ... run the agent
```

---

## Key Resources by Category

| Need | Go-To Resource |
|------|---------------|
| First principles | [OpenAI Harness Engineering](https://openai.com/index/harness-engineering/) |
| Long-running agents | [Anthropic: Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) |
| Context management | [Manus Context Engineering](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) |
| Agent files format | [AGENTS.md spec](https://github.com/agentsmd/agents.md) |
| Production principles | [12 Factor Agents](https://www.humanlayer.dev/blog/12-factor-agents) |
| Evals guide | [OpenAI: Testing Agent Skills](https://developers.openai.com/blog/eval-skills/) |
| Benchmarking | [AppWorld](https://appworld.dev/), [AgentBench](https://github.com/THUDM/AgentBench) |
| Safe autonomy | [Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing) |

---

## Quick-Start Checklist

When setting up a harness for a new coding project:

```
□ Create AGENTS.md (or CLAUDE.md) with:
    □ Repo overview and structure
    □ Key commands (install, build, test, lint)
    □ Coding conventions
    □ What NOT to touch
    □ Verification steps the agent must run

□ Create .agent/ directory with:
    □ init.sh — environment setup + baseline
    □ verify.sh — gates the agent must pass

□ Design tools to be:
    □ Context-efficient (return summaries, not raw output)
    □ Explicit about side effects
    □ Safe by default (dry-run modes where applicable)

□ Set up tracing:
    □ Log all LLM calls and tool calls as JSONL
    □ Include token counts
    □ Checkpoint state at logical boundaries

□ Write specs before handing off:
    □ Acceptance criteria
    □ Technical constraints  
    □ Verification commands
```
```
