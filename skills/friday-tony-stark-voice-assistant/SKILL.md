```markdown
---
name: friday-tony-stark-voice-assistant
description: Build and extend FRIDAY, a Tony Stark-inspired voice AI assistant using FastMCP, LiveKit Agents, Gemini LLM, Sarvam STT, and OpenAI TTS.
triggers:
  - set up friday tony stark voice assistant
  - add a new tool to the friday mcp server
  - configure friday voice agent providers
  - connect friday to livekit room
  - troubleshoot friday mcp server connection
  - switch stt or tts provider in friday agent
  - run friday voice agent with mcp tools
  - extend friday with custom mcp tools
---

# F.R.I.D.A.Y. — Tony Stark Voice Assistant

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

FRIDAY is a two-process AI voice assistant: a **FastMCP server** exposes tools (web search, news, system info) over SSE, and a **LiveKit voice agent** connects microphone → STT → LLM → TTS → speaker in real time, calling MCP tools as needed.

---

## Architecture

```
Microphone ──► STT (Sarvam Saaras v3)
                    │
                    ▼
             LLM (Gemini 2.5 Flash)  ◄──────► MCP Server (FastMCP / SSE :8000)
                    │                              ├─ get_world_news
                    ▼                              ├─ open_world_monitor
             TTS (OpenAI nova)                     ├─ search_web
                    │                              └─ …more tools
                    ▼
             Speaker / LiveKit room
```

Both processes must run simultaneously. The voice agent connects to the MCP server at `http://127.0.0.1:8000/sse`.

---

## Installation

### Prerequisites

- Python ≥ 3.11
- [`uv`](https://github.com/astral-sh/uv)
- A [LiveKit Cloud](https://cloud.livekit.io) project (free tier works)

```bash
git clone https://github.com/SAGAR-TAMANG/friday-tony-stark-demo.git
cd friday-tony-stark-demo
uv sync          # creates .venv and installs all dependencies
cp .env.example .env
# Fill in your API keys in .env
```

---

## Key Commands

| Command | What it does |
|---------|-------------|
| `uv run friday` | Start the FastMCP server on `http://127.0.0.1:8000/sse` |
| `uv run friday_voice` | Start the LiveKit voice agent in dev mode |

**Always start the MCP server first**, then the voice agent.

```bash
# Terminal 1
uv run friday

# Terminal 2
uv run friday_voice
```

Then open [LiveKit Agents Playground](https://agents-playground.livekit.io) and connect to your room.

---

## Environment Variables

Copy `.env.example` to `.env` and set:

```bash
# LiveKit (required)
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=$LIVEKIT_API_KEY
LIVEKIT_API_SECRET=$LIVEKIT_API_SECRET

# LLM — Gemini (default, required)
GOOGLE_API_KEY=$GOOGLE_API_KEY

# STT — Sarvam (default, required)
SARVAM_API_KEY=$SARVAM_API_KEY

# TTS — OpenAI (default, required)
OPENAI_API_KEY=$OPENAI_API_KEY

# Optional providers
GROQ_API_KEY=$GROQ_API_KEY
DEEPGRAM_API_KEY=$DEEPGRAM_API_KEY
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json

# Optional integrations
SUPABASE_URL=$SUPABASE_URL
SUPABASE_API_KEY=$SUPABASE_API_KEY
```

---

## Project Structure

```
friday-tony-stark-demo/
├── server.py           # Entry: starts FastMCP server
├── agent_friday.py     # Entry: starts LiveKit voice agent
├── pyproject.toml
├── .env.example
└── friday/
    ├── config.py       # Env-var loading & app-wide settings
    ├── tools/
    │   ├── __init__.py # Registers all tools with mcp instance
    │   ├── web.py      # search_web, fetch_url, get_world_news, open_world_monitor
    │   ├── system.py   # get_current_time, get_system_info
    │   └── utils.py    # format_json, word_count
    ├── prompts/        # MCP prompt templates
    └── resources/      # MCP resources (friday://info)
```

---

## Adding a New MCP Tool

### Step 1 — Create or open a file in `friday/tools/`

```python
# friday/tools/weather.py
from fastmcp import FastMCP
import httpx

def register(mcp: FastMCP) -> None:

    @mcp.tool()
    async def get_weather(city: str) -> str:
        """Get current weather for a city."""
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                "https://wttr.in/{city}?format=3",
                params={"city": city},
                timeout=10,
            )
            resp.raise_for_status()
            return resp.text
```

### Step 2 — Register in `friday/tools/__init__.py`

```python
# friday/tools/__init__.py
from friday.tools import web, system, utils, weather  # add your module

def register_all(mcp):
    web.register(mcp)
    system.register(mcp)
    utils.register(mcp)
    weather.register(mcp)   # add this line
```

The MCP server picks up the new tool on next `uv run friday`.

---

## Switching Providers

Edit the constants at the top of `agent_friday.py`:

```python
STT_PROVIDER = "sarvam"   # "sarvam" | "whisper"
LLM_PROVIDER = "gemini"   # "gemini" | "openai"
TTS_PROVIDER = "openai"   # "openai" | "sarvam"
```

---

## Voice Agent Internals

The agent follows the LiveKit Agents pattern:

```python
# agent_friday.py (simplified)
from livekit.agents import AutoSubscribe, JobContext, WorkerOptions, cli, llm
from livekit.agents.voice_assistant import VoiceAssistant
from livekit.plugins import openai, google, sarvam
from livekit.agents.mcp import MCPClient

async def entrypoint(ctx: JobContext):
    await ctx.connect(auto_subscribe=AutoSubscribe.AUDIO_ONLY)

    # Connect to the FastMCP server
    mcp_client = MCPClient("http://127.0.0.1:8000/sse")
    tools = await mcp_client.list_tools()

    initial_ctx = llm.ChatContext().append(
        role="system",
        text="You are FRIDAY, Tony Stark's AI assistant. Be concise and helpful.",
    )

    assistant = VoiceAssistant(
        vad=silero.VAD.load(),
        stt=sarvam.STT(model="saaras:v3"),
        llm=google.LLM(model="gemini-2.5-flash"),
        tts=openai.TTS(voice="nova"),
        chat_ctx=initial_ctx,
        tools=tools,
    )

    assistant.start(ctx.room)
    await asyncio.sleep(1)
    await assistant.say("Hello. I am FRIDAY. How can I assist you?", allow_interruptions=True)

def dev():
    # Injects 'dev' CLI flag so you don't type it manually
    import sys
    sys.argv = [sys.argv[0], "dev"]
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

---

## MCP Server Internals

```python
# server.py (simplified)
from fastmcp import FastMCP
from friday.tools import register_all

mcp = FastMCP("FRIDAY", description="Tony Stark's AI backend")
register_all(mcp)

def main():
    mcp.run(transport="sse", host="127.0.0.1", port=8000)
```

---

## Common Patterns

### Pattern: Tool with structured output

```python
# friday/tools/stocks.py
from fastmcp import FastMCP
from pydantic import BaseModel

class StockInfo(BaseModel):
    symbol: str
    price: float
    change_pct: float

def register(mcp: FastMCP) -> None:

    @mcp.tool()
    async def get_stock_price(symbol: str) -> StockInfo:
        """Fetch current stock price and daily change for a ticker symbol."""
        # ... fetch from API ...
        return StockInfo(symbol=symbol.upper(), price=182.34, change_pct=1.2)
```

### Pattern: Tool with error handling

```python
@mcp.tool()
async def search_web(query: str, max_results: int = 5) -> str:
    """Search the web and return summarised results."""
    try:
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.get(
                "https://api.search-provider.com/search",
                params={"q": query, "n": max_results},
                headers={"Authorization": f"Bearer {os.environ['SEARCH_API_KEY']}"},
            )
            resp.raise_for_status()
            results = resp.json()
            return "\n".join(f"- {r['title']}: {r['snippet']}" for r in results["items"])
    except httpx.TimeoutException:
        return "Search timed out. Please try again."
    except httpx.HTTPStatusError as e:
        return f"Search failed with status {e.response.status_code}."
```

### Pattern: Tool that opens a local app or URL

```python
@mcp.tool()
def open_world_monitor() -> str:
    """Open the system monitor application."""
    import subprocess, sys
    if sys.platform == "win32":
        subprocess.Popen(["taskmgr"])
    elif sys.platform == "darwin":
        subprocess.Popen(["open", "-a", "Activity Monitor"])
    else:
        subprocess.Popen(["gnome-system-monitor"])
    return "World monitor opened."
```

### Pattern: MCP prompt template

```python
# friday/prompts/summarize.py
from fastmcp import FastMCP

def register(mcp: FastMCP) -> None:

    @mcp.prompt()
    def summarize(text: str, max_words: int = 100) -> str:
        """Prompt to summarise a block of text."""
        return (
            f"Summarise the following text in {max_words} words or fewer, "
            f"maintaining key points:\n\n{text}"
        )
```

---

## WSL / Windows Host IP

When running inside WSL, the MCP server URL resolves automatically to the Windows host IP. No extra config needed — `agent_friday.py` handles this:

```python
import os, socket

def get_mcp_url() -> str:
    if "microsoft" in open("/proc/version").read().lower():
        # Inside WSL — use Windows host IP
        host = socket.gethostbyname(socket.gethostname() + ".local")
        return f"http://{host}:8000/sse"
    return "http://127.0.0.1:8000/sse"
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Connection refused` on port 8000 | Start `uv run friday` before `uv run friday_voice` |
| Voice agent can't find tools | Verify MCP server is running: `curl http://127.0.0.1:8000/sse` |
| `LIVEKIT_URL` error | Set `LIVEKIT_URL=wss://your-project.livekit.cloud` in `.env` |
| STT not transcribing | Check `SARVAM_API_KEY` is valid; try switching to `STT_PROVIDER = "whisper"` |
| Gemini LLM errors | Ensure `GOOGLE_API_KEY` is set and has Gemini 2.5 Flash access |
| TTS silent output | Verify `OPENAI_API_KEY` is valid and has TTS quota |
| Tool not appearing in LLM | Confirm `register(mcp)` is called in `friday/tools/__init__.py` |
| WSL: MCP URL unreachable | Check Windows Firewall allows port 8000; use host IP instead of `127.0.0.1` |

---

## pyproject.toml Entry Points

The `uv run` shortcuts are defined as script entry points:

```toml
[project.scripts]
friday = "server:main"
friday_voice = "agent_friday:dev"
```

---

## Quick Reference

```bash
# Install
uv sync

# Start MCP server (Terminal 1)
uv run friday

# Start voice agent (Terminal 2)
uv run friday_voice

# Verify MCP server tools are registered
curl -N http://127.0.0.1:8000/sse

# Add a tool: create friday/tools/mytool.py → register(mcp) → add to __init__.py

# Switch providers: edit STT_PROVIDER / LLM_PROVIDER / TTS_PROVIDER in agent_friday.py
```
```
