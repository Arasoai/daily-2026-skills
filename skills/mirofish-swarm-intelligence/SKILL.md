```markdown
---
name: mirofish-swarm-intelligence
description: MiroFish is a multi-agent swarm intelligence engine that builds high-fidelity digital parallel worlds from seed data to simulate and predict social, financial, and narrative outcomes.
triggers:
  - set up MiroFish swarm intelligence
  - simulate multi-agent social prediction
  - run MiroFish prediction engine
  - configure MiroFish agents
  - predict outcomes with swarm intelligence
  - build parallel world simulation
  - use MiroFish for financial forecasting
  - deploy MiroFish multi-agent system
---

# MiroFish Swarm Intelligence Engine

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

MiroFish is a Python-based multi-agent swarm intelligence engine that ingests seed materials (news, policy drafts, financial signals, fiction text) and constructs a high-fidelity digital parallel world. Thousands of agents with distinct personalities, long-term memory, and behavioral logic interact and evolve, enabling deep simulation and prediction of social trends, financial outcomes, public opinion, and narrative continuations. Built on top of OASIS (CAMEL-AI) with a React frontend and Flask backend.

---

## Architecture Overview

```
Seed Material (text/data)
        ↓
[Graph Construction] — Entity extraction, GraphRAG, memory injection
        ↓
[Environment Setup] — Agent persona generation, relationship mapping
        ↓
[Dual-Platform Simulation] — Parallel agent interaction + temporal memory updates
        ↓
[Report Generation] — ReportAgent with toolset queries simulation world
        ↓
[Deep Interaction] — Chat with any agent or ReportAgent
```

**Stack:**
- **Backend:** Python ≥3.11, ≤3.12 (Flask, OASIS engine)
- **Frontend:** Node.js 18+, React (port 3000)
- **Memory:** Zep Cloud (long-term agent memory)
- **LLM:** Any OpenAI-SDK-compatible API (Qwen, GPT-4, etc.)
- **Package manager:** `uv` for Python

---

## Installation

### Prerequisites

| Tool | Version | Check |
|------|---------|-------|
| Node.js | 18+ | `node -v` |
| Python | ≥3.11, ≤3.12 | `python --version` |
| uv | latest | `uv --version` |

Install `uv` if missing:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Clone and Configure

```bash
git clone https://github.com/666ghj/MiroFish.git
cd MiroFish

# Copy environment template
cp .env.example .env
```

### Environment Variables (`.env`)

```env
# LLM — must be OpenAI SDK compatible
# Recommended: Alibaba Qwen via DashScope
LLM_API_KEY=$LLM_API_KEY
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL_NAME=qwen-plus

# Zep Cloud — agent long-term memory
# Free tier at https://app.getzep.com/
ZEP_API_KEY=$ZEP_API_KEY
```

**Other supported LLM providers** (change `LLM_BASE_URL` and `LLM_MODEL_NAME`):
```env
# OpenAI
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL_NAME=gpt-4o

# Local Ollama
LLM_BASE_URL=http://localhost:11434/v1
LLM_MODEL_NAME=llama3.1
LLM_API_KEY=ollama
```

### Install All Dependencies

```bash
# One-command full install (Node + Python)
npm run setup:all
```

Or step-by-step:
```bash
npm run setup           # Node.js deps (root + frontend)
npm run setup:backend   # Python deps via uv (creates .venv automatically)
```

---

## Running MiroFish

### Development Mode (Both Services)

```bash
# From project root — starts frontend (3000) and backend (5001)
npm run dev
```

### Individual Services

```bash
npm run backend    # Flask API on http://localhost:5001
npm run frontend   # React UI on http://localhost:3000
```

### Docker Deployment

```bash
cp .env.example .env
# Edit .env with your keys

docker compose up -d
# Frontend: http://localhost:3000
# Backend:  http://localhost:5001
```

---

## Key Configuration

### Simulation Parameters

When starting a simulation via the UI or API, key parameters include:

| Parameter | Description | Recommended |
|-----------|-------------|-------------|
| `num_agents` | Number of agents in simulation | 50–500 |
| `max_rounds` | Simulation time steps | Start with <40 for testing |
| `seed_material` | Input text/data describing the world | Required |
| `prediction_query` | Natural language prediction question | Required |

> ⚠️ **Cost Note:** Simulations consume significant LLM tokens. Use <40 rounds for initial testing.

### Agent Memory (Zep Cloud)

MiroFish uses Zep for persistent agent memory across simulation rounds:

```python
# Backend uses ZEP_API_KEY from environment
# Each agent gets a unique Zep session for long-term memory
# Memory is updated each round with new interactions
```

---

## Backend API Reference

Base URL: `http://localhost:5001`

### Start a Simulation

```python
import requests

response = requests.post("http://localhost:5001/api/simulation/start", json={
    "seed_material": "Your seed text here — news article, report, or story excerpt",
    "prediction_query": "What will public opinion look like in 30 days?",
    "num_agents": 100,
    "max_rounds": 30
})

simulation_id = response.json()["simulation_id"]
print(f"Simulation started: {simulation_id}")
```

### Check Simulation Status

```python
import requests

status = requests.get(f"http://localhost:5001/api/simulation/{simulation_id}/status")
print(status.json())
# {"status": "running", "current_round": 15, "total_rounds": 30}
```

### Get Prediction Report

```python
report = requests.get(f"http://localhost:5001/api/simulation/{simulation_id}/report")
print(report.json()["report"])
```

### Chat with an Agent

```python
response = requests.post(f"http://localhost:5001/api/simulation/{simulation_id}/chat", json={
    "agent_id": "agent_042",
    "message": "What do you think about the recent policy changes?"
})
print(response.json()["reply"])
```

### Chat with ReportAgent

```python
response = requests.post(f"http://localhost:5001/api/simulation/{simulation_id}/report-agent/chat", json={
    "message": "Summarize the key opinion clusters that emerged in this simulation"
})
print(response.json()["reply"])
```

### List All Agents

```python
agents = requests.get(f"http://localhost:5001/api/simulation/{simulation_id}/agents")
for agent in agents.json()["agents"]:
    print(f"{agent['id']}: {agent['persona']['name']} — {agent['persona']['role']}")
```

---

## Python Backend Integration Examples

### Direct Python Usage (Backend Module)

```python
import asyncio
import os
from backend.engine.simulation import MiroFishSimulation

async def run_prediction():
    sim = MiroFishSimulation(
        llm_api_key=os.environ["LLM_API_KEY"],
        llm_base_url=os.environ["LLM_BASE_URL"],
        llm_model_name=os.environ["LLM_MODEL_NAME"],
        zep_api_key=os.environ["ZEP_API_KEY"]
    )
    
    # Load seed material
    with open("my_report.txt", "r") as f:
        seed_text = f.read()
    
    # Build the knowledge graph and agent world
    await sim.initialize(
        seed_material=seed_text,
        num_agents=100
    )
    
    # Run simulation
    result = await sim.run(
        prediction_query="How will public opinion evolve over the next month?",
        max_rounds=30
    )
    
    print(result.report)
    return result

asyncio.run(run_prediction())
```

### Custom Agent Persona Definition

```python
from backend.engine.agent import AgentPersona

# Define a custom agent persona
persona = AgentPersona(
    name="Zhang Wei",
    age=34,
    occupation="Software Engineer",
    political_leaning="moderate",
    personality_traits=["analytical", "skeptical", "tech-savvy"],
    background="Lives in Shanghai, follows tech news closely",
    initial_beliefs={
        "topic_stance": "cautiously optimistic",
        "trust_in_media": 0.6
    }
)
```

### Knowledge Graph Construction

```python
from backend.engine.graph import KnowledgeGraphBuilder

builder = KnowledgeGraphBuilder(
    llm_api_key=os.environ["LLM_API_KEY"],
    llm_base_url=os.environ["LLM_BASE_URL"],
    llm_model_name=os.environ["LLM_MODEL_NAME"]
)

# Extract entities and relationships from seed material
graph = await builder.build(
    text=seed_text,
    extract_entities=True,
    extract_relationships=True,
    build_graphrag=True
)

print(f"Entities: {len(graph.entities)}")
print(f"Relationships: {len(graph.relationships)}")
```

### Inject Variables Mid-Simulation (God Mode)

```python
# Dynamically inject an event into a running simulation
await sim.inject_event(
    round_number=15,
    event_description="A major earthquake strikes the region, causing infrastructure disruption",
    affected_agents="all",  # or list of agent IDs
    intensity=0.8
)
```

### Accessing Agent Memory via Zep

```python
from zep_cloud.client import AsyncZep
import os

zep = AsyncZep(api_key=os.environ["ZEP_API_KEY"])

# Retrieve an agent's memory across simulation rounds
memory = await zep.memory.get(session_id="agent_042_session_id")
for message in memory.messages:
    print(f"[Round {message.metadata.get('round')}] {message.content}")
```

---

## Common Workflow Patterns

### Pattern 1: Public Opinion Analysis (舆情推演)

```python
# 1. Prepare seed material — paste news reports, social media summaries
seed = """
武汉大学樱花季游客管理问题引发热议。
多名学生反映校园人流量过大，影响正常教学秩序...
[Full report text]
"""

# 2. Start simulation focused on opinion dynamics
result = await sim.initialize_and_run(
    seed_material=seed,
    prediction_query="未来两周内，舆论走向如何变化？主要意见群体有哪些？",
    num_agents=200,
    max_rounds=40,
    simulation_type="public_opinion"
)
```

### Pattern 2: Financial Market Prediction

```python
seed = """
Q3 earnings report shows 23% YoY revenue growth.
Federal Reserve signals potential rate cuts in Q1.
Major institutional investors increasing positions...
"""

result = await sim.initialize_and_run(
    seed_material=seed,
    prediction_query="What is the likely market sentiment and price movement over the next 2 weeks?",
    num_agents=150,
    max_rounds=30,
    simulation_type="financial"
)
```

### Pattern 3: Narrative Continuation (小说结局推演)

```python
# Load the first 80 chapters of a novel
with open("dream_of_red_chamber_ch1_80.txt", "r") as f:
    novel_text = f.read()

result = await sim.initialize_and_run(
    seed_material=novel_text,
    prediction_query="Based on character relationships and plot arcs, predict the ending for the main characters",
    num_agents=50,  # key characters as agents
    max_rounds=60,
    simulation_type="narrative"
)
```

### Pattern 4: Policy Impact Analysis

```python
seed = """
Draft policy: New carbon tax of $50/ton to be implemented Q2 2026.
Current industrial output data, employment figures, and energy mix...
"""

result = await sim.initialize_and_run(
    seed_material=seed,
    prediction_query="How will different demographic and industry groups respond to this policy?",
    num_agents=300,
    max_rounds=50,
    simulation_type="social_prediction"
)
```

---

## Project Structure

```
MiroFish/
├── .env.example              # Environment variable template
├── package.json              # Root npm scripts
├── docker-compose.yml        # Docker deployment config
├── frontend/                 # React application
│   ├── src/
│   │   ├── components/       # UI components
│   │   └── pages/            # Simulation, Report, Chat pages
│   └── package.json
├── backend/                  # Python Flask API
│   ├── app.py                # Flask entry point (port 5001)
│   ├── pyproject.toml        # Python dependencies (uv)
│   ├── engine/
│   │   ├── simulation.py     # Core simulation orchestrator
│   │   ├── agent.py          # Agent persona & behavior
│   │   ├── graph.py          # Knowledge graph / GraphRAG
│   │   ├── memory.py         # Zep memory integration
│   │   └── report.py         # ReportAgent with toolset
│   └── api/
│       └── routes.py         # API endpoint definitions
└── static/
    └── image/                # Screenshots and assets
```

---

## Troubleshooting

### Python Version Issues

```bash
# MiroFish requires Python >=3.11, <=3.12 strictly
python --version  # Must be 3.11.x or 3.12.x

# Use uv to pin the version
uv python install 3.12
uv python pin 3.12
npm run setup:backend
```

### Backend Won't Start (Port 5001 Conflict)

```bash
# Find and kill process on 5001
lsof -ti:5001 | xargs kill -9

# Or change port in backend config
# Set BACKEND_PORT=5002 in .env
```

### Zep Memory Connection Errors

```bash
# Verify ZEP_API_KEY is set
echo $ZEP_API_KEY

# Test Zep connection
python -c "
import asyncio
from zep_cloud.client import AsyncZep
import os
async def test():
    zep = AsyncZep(api_key=os.environ['ZEP_API_KEY'])
    print('Zep connected successfully')
asyncio.run(test())
"
```

### LLM API Errors / Token Limits

```bash
# Reduce simulation scale for testing
# In your API call or .env:
MAX_ROUNDS=20       # Start small
NUM_AGENTS=50       # Start small

# Check model availability
python -c "
import openai, os
client = openai.OpenAI(
    api_key=os.environ['LLM_API_KEY'],
    base_url=os.environ['LLM_BASE_URL']
)
models = client.models.list()
print([m.id for m in models.data[:5]])
"
```

### Docker: Images Slow to Pull (China)

```yaml
# In docker-compose.yml, uncomment accelerated mirror addresses
# provided in the comments of the file for faster pulls in China
```

### Frontend Cannot Reach Backend

```bash
# Verify both services are running
curl http://localhost:5001/api/health
curl http://localhost:3000

# Check CORS config — backend should allow localhost:3000
# Verify FRONTEND_URL=http://localhost:3000 in .env if configurable
```

### Simulation Runs Out of Memory

```bash
# Large simulations (500+ agents, 100+ rounds) are memory intensive
# Monitor usage:
watch -n 2 'ps aux | grep python | grep -v grep'

# Reduce batch size in backend config if available
AGENT_BATCH_SIZE=20  # Process agents in smaller batches
```

---

## Cost Estimation

| Simulation Size | Agents | Rounds | Approx. Tokens | Approx. Cost (qwen-plus) |
|-----------------|--------|--------|-----------------|--------------------------|
| Small (test) | 50 | 20 | ~500K | ~¥5 |
| Medium | 100 | 40 | ~2M | ~¥20 |
| Large | 300 | 60 | ~8M | ~¥80 |
| Full scale | 500 | 100 | ~20M | ~¥200 |

> Always start with small simulations to validate your seed material and prediction query.

---

## Resources

- **Live Demo:** https://666ghj.github.io/mirofish-demo/
- **Homepage:** https://mirofish.ai
- **OASIS Engine:** https://github.com/camel-ai/oasis
- **Zep Cloud:** https://app.getzep.com/
- **Alibaba DashScope (Qwen):** https://bailian.console.aliyun.com/
- **Discord:** https://discord.com/channels/1469200078932545606/1469201282077163739
- **DeepWiki Docs:** https://deepwiki.com/666ghj/MiroFish
- **Contact:** mirofish@shanda.com
```
