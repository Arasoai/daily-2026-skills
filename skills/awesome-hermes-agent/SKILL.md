```markdown
---
name: awesome-hermes-agent
description: Curated ecosystem guide for Hermes Agent by Nous Research — a self-improving AI agent with skills, memory, multi-platform messaging, and MCP integration
triggers:
  - "help me set up Hermes Agent"
  - "how do I install skills for Hermes"
  - "configure Hermes Agent for my project"
  - "add a skill to Hermes"
  - "Hermes Agent deployment and integrations"
  - "use Hermes with Telegram or Discord"
  - "what skills are available for Hermes Agent"
  - "Nous Research Hermes Agent workflow"
---

# Awesome Hermes Agent

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated ecosystem of skills, tools, integrations, and resources for [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research — the self-improving AI agent with a closed learning loop, multi-platform gateway support, and a growing skills ecosystem.

---

## What Is Hermes Agent?

Hermes Agent is a self-improving AI agent that:

- **Creates skills from experience** — learns and stores reusable capabilities as procedural memory
- **Improves skills during use** — refines skills automatically through feedback loops
- **Searches past conversations** — episodic memory across sessions
- **Builds a model of you** — deepening user context over time
- **Runs anywhere** — $5 VPS, GPU cluster, or serverless; talk to it via Telegram while it works on a cloud VM
- **Supports MCP** — integrates with Model Context Protocol tool servers
- **Schedules tasks** — built-in cron scheduling
- **Multi-platform messaging** — Telegram, Discord, Slack, WhatsApp, Signal

---

## Installation

### Quick Start (Official)

Follow the [Official Docs quickstart](https://hermes-agent.nousresearch.com/docs/) for the authoritative installation guide.

```bash
# Clone the core project
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent

# Install dependencies (Python-based)
pip install -e .

# Or with uv (faster)
uv pip install -e .
```

### Environment Configuration

```bash
# Copy example config
cp .env.example .env

# Required: Set your model provider API key
export OPENAI_API_KEY=your_key_here        # or
export ANTHROPIC_API_KEY=your_key_here    # or
export OPENROUTER_API_KEY=your_key_here

# Optional: Telegram gateway
export TELEGRAM_BOT_TOKEN=your_token_here

# Optional: Discord gateway
export DISCORD_BOT_TOKEN=your_token_here
```

### Minimal `.env` Example

```env
# Model backend
OPENROUTER_API_KEY=sk-or-...
DEFAULT_MODEL=openai/gpt-4o

# Memory / skills storage
HERMES_DATA_DIR=~/.hermes

# Messaging gateway (pick one or more)
TELEGRAM_BOT_TOKEN=...
DISCORD_BOT_TOKEN=...
```

---

## Key CLI Commands

```bash
# Start an interactive session
hermes chat

# Run in daemon mode (background, messaging gateway active)
hermes serve

# List installed skills
hermes skills list

# Install a skill from a path or URL
hermes skills install ./my-skill
hermes skills install https://github.com/user/repo

# Run a single task non-interactively
hermes run "summarize the last 10 git commits"

# Search conversation memory
hermes memory search "database migration"

# Show agent status
hermes status

# Cron: list scheduled tasks
hermes cron list

# Cron: add a task
hermes cron add "0 9 * * *" "send me a daily briefing"
```

---

## Skills Architecture

Skills are the core of the Hermes learning loop. A skill is a reusable capability stored as structured memory.

### Skill File Format (`SKILL.md`)

```markdown
---
name: my-skill
description: Does X given Y
triggers:
  - "do X"
  - "help me with Y"
---

# My Skill

## Instructions
...step-by-step instructions the agent follows...

## Examples
...worked examples...
```

### Installing Community Skills

```bash
# Install wondelai/skills (cross-platform, 250+ skills)
git clone https://github.com/wondelai/skills ~/.hermes/skills/wondelai
hermes skills reload

# Install cybersecurity skills (MITRE ATT&CK mapped)
git clone https://github.com/mukul975/Anthropic-Cybersecurity-Skills ~/.hermes/skills/cybersec
hermes skills reload
```

### Creating a Skill Programmatically

```python
from hermes_agent import HermesAgent, Skill

agent = HermesAgent()

# Define a new skill
skill = Skill(
    name="git-summarize",
    description="Summarizes recent git activity",
    triggers=["summarize git", "what changed recently", "git digest"],
    instructions="""
    1. Run `git log --oneline -20`
    2. Group commits by type (feat, fix, chore, etc.)
    3. Return a concise bullet-point summary
    """,
)

agent.skills.register(skill)
agent.skills.save()  # persists to HERMES_DATA_DIR
```

---

## Core Python API

```python
from hermes_agent import HermesAgent

# Initialize agent (picks up .env automatically)
agent = HermesAgent()

# Single-turn query
response = await agent.chat("What files did I work on yesterday?")
print(response.text)

# Multi-turn conversation
session = agent.new_session()
r1 = await session.send("I'm building a FastAPI app")
r2 = await session.send("Add authentication to it")  # has context from r1

# Access memory
memories = agent.memory.search("FastAPI authentication")
for m in memories:
    print(m.summary, m.created_at)

# Trigger a skill explicitly
result = await agent.skills.run("git-summarize", context={"repo": "/path/to/repo"})

# Schedule a cron task
agent.cron.add(
    schedule="0 8 * * 1-5",  # weekdays at 8am
    task="send me a standup prompt with yesterday's git activity",
)
```

---

## MCP Integration

Hermes supports Model Context Protocol servers as tool backends.

```python
from hermes_agent import HermesAgent
from hermes_agent.mcp import MCPServer

agent = HermesAgent()

# Register an MCP server
agent.mcp.register(
    MCPServer(
        name="filesystem",
        command="npx",
        args=["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"],
    )
)

# The agent can now use filesystem MCP tools in any conversation
response = await agent.chat("List all Python files in my projects folder")
```

```yaml
# Or configure in hermes.yaml
mcp:
  servers:
    - name: filesystem
      command: npx
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
    - name: github
      command: npx
      args: ["-y", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"
```

---

## Configuration Reference (`hermes.yaml`)

```yaml
# hermes.yaml — place in project root or ~/.hermes/

model:
  default: openai/gpt-4o
  fallback: openai/gpt-4o-mini
  temperature: 0.7

memory:
  backend: sqlite          # sqlite | postgres | chroma
  data_dir: ~/.hermes
  max_context_turns: 50

skills:
  directories:
    - ~/.hermes/skills
    - ./skills
  auto_improve: true       # refine skills after use

gateway:
  telegram:
    enabled: true
    token: "${TELEGRAM_BOT_TOKEN}"
    allowed_users: ["your_telegram_id"]
  discord:
    enabled: false
    token: "${DISCORD_BOT_TOKEN}"

cron:
  enabled: true
  timezone: America/New_York

terminal:
  backend: local           # local | docker | e2b | modal | ssh | k8s
```

---

## Deployment Patterns

### Local Development

```bash
# Simple local run
hermes chat

# With a specific model
HERMES_MODEL=anthropic/claude-3-5-sonnet hermes chat
```

### Docker

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install hermes-agent

ENV HERMES_DATA_DIR=/data
VOLUME ["/data"]

CMD ["hermes", "serve"]
```

```bash
docker run -d \
  -e OPENROUTER_API_KEY=$OPENROUTER_API_KEY \
  -e TELEGRAM_BOT_TOKEN=$TELEGRAM_BOT_TOKEN \
  -v hermes-data:/data \
  my-hermes-agent
```

### Serverless (Modal)

```python
import modal
from hermes_agent import HermesAgent

app = modal.App("hermes-agent")

@app.function(
    secrets=[modal.Secret.from_name("hermes-secrets")],
    timeout=300,
)
async def run_task(prompt: str) -> str:
    agent = HermesAgent()
    response = await agent.chat(prompt)
    return response.text
```

### Remote Terminal Backend (SSH to cloud VM)

```yaml
# hermes.yaml
terminal:
  backend: ssh
  ssh:
    host: your-vm.example.com
    user: ubuntu
    key_path: ~/.ssh/id_ed25519
```

---

## Multi-Agent Patterns

### Using mission-control for Fleet Management

```bash
# Install mission-control (3k+ stars, broad agent orchestration)
git clone https://github.com/builderz-labs/mission-control
cd mission-control && npm install && npm run dev
# Then register your Hermes instance in the dashboard
```

### Inter-Agent Bridge (hermes-plugins)

```bash
# Install hermes-plugins for inter-agent communication
git clone https://github.com/42-evey/hermes-plugins ~/.hermes/plugins/42-evey
hermes plugins reload
```

```python
# Send a task from one Hermes instance to another
from hermes_agent.plugins import InterAgentBridge

bridge = InterAgentBridge(target_agent_url="http://agent2:8080")
result = await bridge.delegate("Refactor the authentication module")
```

---

## The agentskills.io Standard

Many skills in the ecosystem follow the [agentskills.io](https://agentskills.io) open standard, making them portable across Hermes, Claude Code, Cursor, Codex, and others.

### Installing agentskills.io Skills

```bash
# Cross-platform skills library
git clone https://github.com/wondelai/skills ~/.hermes/skills/wondelai

# Chainlink oracle skills (official)
git clone https://github.com/smartcontractkit/chainlink-agent-skills ~/.hermes/skills/chainlink

# FLUX image generation (official Black Forest Labs)
git clone https://github.com/black-forest-labs/skills ~/.hermes/skills/flux

# Pydantic AI type-safe skills
pip install pydantic-ai-skills
```

### Writing an agentskills.io-Compliant Skill

```python
# skill.py — agentskills.io compliant
from agentskills import skill, SkillInput, SkillOutput
from pydantic import BaseModel

class SearchInput(SkillInput):
    query: str
    max_results: int = 10

class SearchOutput(SkillOutput):
    results: list[dict]
    total: int

@skill(
    name="web-search",
    description="Search the web and return structured results",
    triggers=["search for", "look up", "find information about"],
)
async def web_search(input: SearchInput) -> SearchOutput:
    # implementation
    results = await do_search(input.query, input.max_results)
    return SearchOutput(results=results, total=len(results))
```

---

## Common Patterns

### Pattern: Skill Auto-Generation

Use `hermes-skill-factory` to create skills from repeated workflows:

```bash
git clone https://github.com/Romanescu11/hermes-skill-factory ~/.hermes/skills/skill-factory
hermes skills reload

# Now in chat:
# "Create a skill for the way I deploy to production"
# The agent observes your workflow and writes a skill for it
```

### Pattern: SRE / Incident Response

```bash
# Install hermes-incident-commander
git clone https://github.com/Lethe044/hermes-incident-commander ~/.hermes/skills/incident

# In hermes.yaml — add monitoring cron
```

```yaml
cron:
  tasks:
    - schedule: "*/5 * * * *"
      task: "check all production services for anomalies and alert if any are degraded"
```

### Pattern: Weather + ML Pipeline

```bash
# Install weather plugin
git clone https://github.com/FahrenheitResearch/hermes-weather-plugin ~/.hermes/plugins/weather

# Usage in chat
# "Show me the NEXRAD radar for the last 6 hours over Chicago"
# "Build a training dataset from HRRR model data for last week"
```

### Pattern: Literate Programming

```bash
pip install litprog-skill
hermes skills reload

# Now ask Hermes to write documented, executable notebooks
# "Write a literate program explaining this authentication module"
```

---

## Self-Evolution Pipeline

For research/advanced use — evolve Hermes's own prompts using DSPy and GEPA:

```bash
git clone https://github.com/NousResearch/hermes-agent-self-evolution
cd hermes-agent-self-evolution
pip install -e .

# Run an evolution cycle
python evolve.py \
  --base-model openai/gpt-4o \
  --eval-suite ./evals/coding \
  --generations 10 \
  --output ./evolved-prompts
```

---

## Troubleshooting

### Skills Not Loading

```bash
# Check skill directories are configured
hermes config show | grep skills

# Validate a specific skill file
hermes skills validate ./my-skill/SKILL.md

# Force reload all skills
hermes skills reload --force

# Check skill logs
hermes logs --filter skills
```

### Memory / Context Issues

```bash
# Check memory backend status
hermes memory status

# Compact/vacuum SQLite memory store
hermes memory compact

# Export memory for debugging
hermes memory export --output memory-dump.json

# Clear a specific session
hermes memory clear --session SESSION_ID
```

### Messaging Gateway Not Receiving Messages

```bash
# Test Telegram connectivity
hermes gateway test telegram

# Check webhook status
hermes gateway status

# Re-register webhook
hermes gateway restart telegram
```

### MCP Server Connection Failures

```bash
# List MCP servers and their status
hermes mcp list

# Test a specific MCP server
hermes mcp test filesystem

# View MCP server logs
hermes logs --filter mcp --server filesystem
```

### Agent Loops / Hangs

```bash
# Set a max turn limit
hermes chat --max-turns 20

# Enable verbose mode to see what's happening
hermes chat --verbose

# Kill a runaway agent process
hermes status       # find the session ID
hermes kill SESSION_ID
```

---

## Key Resources

| Resource | URL |
|----------|-----|
| Official Docs | https://hermes-agent.nousresearch.com/docs/ |
| Core Repo | https://github.com/NousResearch/hermes-agent |
| Skills Standard | https://agentskills.io |
| Discord Community | https://discord.gg/NousResearch |
| Community Skills | https://github.com/wondelai/skills |
| Cybersecurity Skills | https://github.com/mukul975/Anthropic-Cybersecurity-Skills |
| GUI Dashboard | https://github.com/builderz-labs/mission-control |
| Self-Evolution | https://github.com/NousResearch/hermes-agent-self-evolution |
```
