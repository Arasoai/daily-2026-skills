```markdown
---
name: hermes-agent-nous-research
description: Practical guide skill for Hermes Agent, the open-source AI Agent framework by Nous Research featuring a self-improving learning loop, three-layer memory system, and automatic Skill creation and evolution.
triggers:
  - "set up hermes agent"
  - "use hermes agent framework"
  - "configure hermes agent skills"
  - "hermes agent memory system"
  - "nous research hermes agent"
  - "build agent with hermes"
  - "hermes agent learning loop"
  - "hermes agent tool integration"
---

# Hermes Agent — Nous Research Framework Guide

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Hermes Agent is an open-source AI Agent framework by [Nous Research](https://hermes-agent.nousresearch.com/) (released February 2026). It differentiates from Claude Code and OpenClaw through three core innovations:

1. **Self-Improving Learning Loop** — the agent observes outcomes and refines its own behavior
2. **Three-Layer Memory System** — working memory, episodic memory, and semantic/skill memory
3. **Automatic Skill Creation & Evolution** — reusable Skill modules are generated and improved from experience

Official repo: [https://github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)  
Docs: [https://hermes-agent.nousresearch.com/docs/](https://hermes-agent.nousresearch.com/docs/)

---

## Installation

### Prerequisites

- Python 3.10+
- Node.js 18+ (for web-based integrations)
- An LLM API key (OpenAI, Anthropic, or a local model via Ollama)

### Install via pip

```bash
pip install hermes-agent
```

### Install from source

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
pip install -e ".[dev]"
```

### Verify installation

```bash
hermes --version
hermes doctor  # checks dependencies and config
```

---

## Configuration

Hermes Agent uses a config file at `~/.hermes/config.yaml` (auto-created on first run) and respects environment variables.

### Environment Variables

```bash
# LLM provider credentials
export HERMES_LLM_PROVIDER=openai          # openai | anthropic | ollama | openrouter
export OPENAI_API_KEY=your_key_here
export ANTHROPIC_API_KEY=your_key_here

# Memory backend (default: local sqlite)
export HERMES_MEMORY_BACKEND=sqlite        # sqlite | postgres | redis
export HERMES_MEMORY_PATH=~/.hermes/memory.db

# Skill registry
export HERMES_SKILL_REGISTRY=~/.hermes/skills/
export HERMES_AUTO_SKILL_CREATION=true

# Logging
export HERMES_LOG_LEVEL=info               # debug | info | warn | error
```

### `~/.hermes/config.yaml` structure

```yaml
llm:
  provider: openai
  model: gpt-4o                # or claude-3-7-sonnet, hermes-3-70b, etc.
  temperature: 0.2
  max_tokens: 8192

memory:
  backend: sqlite
  path: ~/.hermes/memory.db
  working_memory_ttl: 3600     # seconds; ephemeral per-session context
  episodic_retention_days: 90  # how long to keep past session logs

skills:
  registry: ~/.hermes/skills/
  auto_create: true            # agent can write new Skills from experience
  auto_evolve: true            # agent can improve existing Skills

tools:
  web_search: true
  code_execution: true
  file_system: true
  shell: false                 # disable for sandboxed environments

harness:
  instructions_path: ~/.hermes/instructions.md
  constraints_path: ~/.hermes/constraints.md
```

---

## CLI Key Commands

```bash
# Start interactive agent session
hermes chat

# Run a one-shot task
hermes run "Summarize the latest commits in this repo"

# Run with a specific skill loaded
hermes run --skill python-refactor "Refactor src/utils.py for readability"

# List installed skills
hermes skills list

# Install a skill from the registry
hermes skills install python-refactor

# Create a new skill interactively
hermes skills create

# Inspect memory
hermes memory show --type episodic --last 10
hermes memory show --type semantic

# Clear working memory (keeps episodic + semantic)
hermes memory clear --working

# Export all memory to JSON
hermes memory export --output memory-backup.json

# Show agent's self-evaluation log (learning loop output)
hermes log --type learning --last 20

# Doctor / diagnostics
hermes doctor

# Update hermes agent
hermes update
```

---

## Python SDK — Core Usage Patterns

### Basic Agent Session

```python
from hermes_agent import HermesAgent

agent = HermesAgent(
    provider="openai",
    model="gpt-4o",
    # API key read from OPENAI_API_KEY env var automatically
)

response = agent.run("List all Python files in the current directory and summarize their purpose.")
print(response.output)
print(response.skills_used)   # Skills the agent invoked
print(response.memory_refs)   # Memory entries accessed
```

### Streaming Responses

```python
from hermes_agent import HermesAgent

agent = HermesAgent(provider="anthropic", model="claude-3-7-sonnet")

for chunk in agent.stream("Write and explain a binary search implementation in Python"):
    print(chunk.text, end="", flush=True)
```

### Using the Three-Layer Memory System

```python
from hermes_agent import HermesAgent
from hermes_agent.memory import MemoryLayer

agent = HermesAgent()

# Write to semantic memory (persistent facts/skills)
agent.memory.write(
    layer=MemoryLayer.SEMANTIC,
    key="project_context",
    value="This is a FastAPI app using PostgreSQL and deployed on Fly.io"
)

# Write to episodic memory (past event log)
agent.memory.write(
    layer=MemoryLayer.EPISODIC,
    content="Refactored the auth module on 2026-04-08, moved JWT logic to services/auth.py"
)

# Read from memory
context = agent.memory.read(layer=MemoryLayer.SEMANTIC, key="project_context")
recent_episodes = agent.memory.search(
    layer=MemoryLayer.EPISODIC,
    query="auth refactor",
    top_k=5
)

# Working memory is managed automatically per session
# but you can inject context manually:
agent.memory.inject_working(
    "The user prefers concise answers with code examples only, no prose explanation."
)
```

### Creating and Using Skills Programmatically

```python
from hermes_agent import HermesAgent
from hermes_agent.skills import Skill, SkillRegistry

# Define a custom Skill
class GitCommitSummarySkill(Skill):
    name = "git-commit-summary"
    description = "Summarizes recent git commits in a readable changelog format"
    version = "1.0.0"

    def run(self, agent, context: dict) -> str:
        num_commits = context.get("num_commits", 10)
        result = agent.tools.shell(f"git log --oneline -{num_commits}")
        return agent.llm.complete(
            f"Format these git commits as a concise changelog:\n{result}"
        )

# Register and use the skill
registry = SkillRegistry()
registry.register(GitCommitSummarySkill())

agent = HermesAgent(skill_registry=registry)
response = agent.run(
    "Summarize the last 20 commits",
    skill_hint="git-commit-summary"
)
print(response.output)
```

### Auto Skill Creation (Self-Improvement Loop)

```python
from hermes_agent import HermesAgent

agent = HermesAgent(
    auto_skill_creation=True,   # agent writes Skills when it detects repetitive patterns
    auto_evolve=True,           # agent improves existing Skills based on outcome feedback
)

# The agent will observe that it repeatedly does this pattern
# and may auto-generate a "dependency-audit" Skill after a few runs
for project_path in ["./project-a", "./project-b", "./project-c"]:
    agent.run(f"Audit {project_path} for outdated Python dependencies and suggest upgrades")

# Inspect what Skills were auto-created
new_skills = agent.skills.list(source="auto-created")
for skill in new_skills:
    print(f"{skill.name} v{skill.version} — {skill.description}")
    print(skill.source_code)
```

### Multi-Agent Orchestration

```python
from hermes_agent import HermesAgent, AgentOrchestrator

# Specialist agents
researcher = HermesAgent(
    role="researcher",
    system_prompt="You are a research specialist. Gather facts and sources."
)

writer = HermesAgent(
    role="writer",
    system_prompt="You are a technical writer. Produce clear, structured documentation."
)

reviewer = HermesAgent(
    role="reviewer",
    system_prompt="You are a code reviewer. Identify bugs and improvement areas."
)

# Orchestrator routes tasks
orchestrator = AgentOrchestrator(
    agents=[researcher, writer, reviewer],
    routing_strategy="capability-match"  # or "round-robin", "explicit"
)

result = orchestrator.run(
    "Research the new Python 3.14 features, then write a migration guide, then review it."
)
print(result.final_output)
print(result.agent_trace)  # full chain of which agent handled what
```

### Configuring the Harness (Instructions + Constraints)

```python
from hermes_agent import HermesAgent

agent = HermesAgent(
    instructions="""
    You are a senior Python developer assistant.
    Always prefer stdlib solutions before suggesting third-party packages.
    Write tests alongside any code you produce.
    """,
    constraints="""
    Never delete files without explicit user confirmation.
    Never execute shell commands that modify system state without showing the command first.
    Keep responses under 500 words unless the user asks for more detail.
    """,
)

response = agent.run("Add input validation to the user registration endpoint")
print(response.output)
```

---

## Real-World Scenario Patterns

### 1. Personal Knowledge Assistant

```python
from hermes_agent import HermesAgent

agent = HermesAgent()

# Ingest documents into semantic memory
agent.memory.ingest_documents(
    paths=["./notes/", "./research-papers/"],
    chunk_size=512
)

# Query against ingested knowledge
answer = agent.run("What were the key conclusions from last quarter's research notes?")
print(answer.output)
```

### 2. Development Automation

```python
from hermes_agent import HermesAgent

agent = HermesAgent(tools={"shell": True, "file_system": True, "code_execution": True})

# Automated code review + fix cycle
result = agent.run(
    "Run the test suite, identify failing tests, fix the root cause, and re-run to confirm."
)
print(result.output)
print(result.files_modified)
```

### 3. Content Creation Pipeline

```python
from hermes_agent import HermesAgent, AgentOrchestrator

outline_agent = HermesAgent(role="outliner")
draft_agent = HermesAgent(role="drafter")
edit_agent = HermesAgent(role="editor")

pipeline = AgentOrchestrator(
    agents=[outline_agent, draft_agent, edit_agent],
    routing_strategy="sequential"
)

article = pipeline.run(
    topic="Practical uses of AI Agents in solo developer workflows",
    format="blog post, 1200 words, technical audience"
)
print(article.final_output)
```

---

## SKILL.md Format (for installing custom skills into Hermes)

Hermes reads `SKILL.md` files to extend agent capabilities. Place them in `~/.hermes/skills/` or the project root.

```markdown
---
name: python-refactor
description: Refactors Python files for readability and PEP8 compliance
triggers:
  - refactor this python file
  - clean up my python code
  - improve code readability
---

# Python Refactor Skill

## What it does
Analyzes Python files using AST inspection and applies PEP8, type hint improvements,
and readability fixes. Runs ruff and black as post-processing.

## Steps
1. Read target file(s)
2. Analyze structure with ast module
3. Apply transformations
4. Run `ruff --fix` and `black`
5. Show diff before writing

## Constraints
- Always show diff before writing changes
- Never remove docstrings
- Preserve all existing tests
```

---

## Troubleshooting

### Agent not finding skills
```bash
hermes skills list          # verify skills are registered
hermes doctor               # checks skill registry path
export HERMES_SKILL_REGISTRY=/correct/path/to/skills/
```

### Memory not persisting between sessions
```bash
# Check memory backend config
hermes memory show --type semantic
# If empty, ensure HERMES_MEMORY_PATH points to a writable location
hermes config show | grep memory
```

### LLM API errors
```bash
# Verify credentials are set
hermes doctor
# Switch provider
hermes config set llm.provider anthropic
export ANTHROPIC_API_KEY=your_key_here
```

### Auto-created skills behaving unexpectedly
```python
# Disable auto-creation temporarily and inspect
agent = HermesAgent(auto_skill_creation=False, auto_evolve=False)

# Review auto-created skills before enabling again
skills = agent.skills.list(source="auto-created")
for s in skills:
    print(s.name, s.source_code)
```

### High memory / slow responses
```bash
# Reduce episodic retention window
hermes config set memory.episodic_retention_days 30
hermes memory prune --older-than 30d
```

---

## Key Concepts Summary

| Concept | Description |
|---|---|
| **Learning Loop** | Agent evaluates task outcomes and updates its behavior model |
| **Working Memory** | Ephemeral per-session context (auto-managed) |
| **Episodic Memory** | Timestamped log of past agent actions and outcomes |
| **Semantic Memory** | Persistent facts, project context, and distilled knowledge |
| **Skill** | Reusable capability module (can be human-written or auto-generated) |
| **Harness** | The five-layer control structure: instructions / constraints / feedback / memory / orchestration |
| **Orchestrator** | Coordinates multiple specialist agents for complex pipelines |

---

## Resources

- [Official Docs](https://hermes-agent.nousresearch.com/docs/)
- [GitHub Repo](https://github.com/NousResearch/hermes-agent)
- [Hermes Agent Orange Book (PDF)](https://github.com/alchaincyf/hermes-agent-orange-book)
- [Nous Research](https://nousresearch.com/)
```
