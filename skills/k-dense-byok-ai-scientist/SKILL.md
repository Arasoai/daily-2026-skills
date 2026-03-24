```markdown
---
name: k-dense-byok-ai-scientist
description: AI co-scientist desktop app with Claude Scientific Skills, multi-model support, and 250+ scientific databases via BYOK API keys
triggers:
  - set up K-Dense BYOK
  - configure Kady AI assistant
  - add scientific skills to my project
  - integrate K-Dense with my research workflow
  - use K-Dense BYOK with my API keys
  - run AI co-scientist locally
  - connect OpenRouter to K-Dense
  - add web search to Kady agent
---

# K-Dense BYOK AI Co-Scientist

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

K-Dense BYOK is an open-source desktop AI research assistant ("Kady") that runs locally, uses your own API keys (BYOK), and gives you access to 170+ scientific skills, 250+ scientific databases, and 500k+ Python packages. It supports 40+ AI models via OpenRouter and optionally delegates heavy compute to Modal cloud infrastructure.

## Architecture

Three services run together on your machine:

| Service | Port | Role |
|---------|------|------|
| Frontend (Next.js) | 3000 | Chat UI, file browser, file preview |
| Backend (FastAPI) | 8000 | Kady agent, expert coordination, tools |
| LiteLLM proxy | 4000 | Routes AI requests to OpenRouter models |

Expert/coding tasks always use Gemini CLI via OpenRouter regardless of the model selected in the UI dropdown.

## Prerequisites

- macOS, Linux, or Windows (via WSL)
- Git
- An OpenRouter API key — [openrouter.ai](https://openrouter.ai/)
- *(Optional)* Parallel API key for web search — [parallel.ai](https://parallel.ai/)
- *(Optional)* Modal credentials for remote compute — [modal.com](https://modal.com/)

## Installation

```bash
git clone https://github.com/K-Dense-AI/k-dense-byok.git
cd k-dense-byok
```

## Configuration

Create `kady_agent/.env`:

```env
# Required
OPENROUTER_API_KEY=$OPENROUTER_API_KEY

# Optional — enables web search
PARALLEL_API_KEY=$PARALLEL_API_KEY

# Internal service wiring — leave as-is
GOOGLE_GEMINI_BASE_URL=http://localhost:4000
GEMINI_API_KEY=sk-litellm-local

# Default model for Kady (changeable in UI dropdown)
DEFAULT_AGENT_MODEL=openrouter/google/gemini-3.1-pro-preview

# Optional — Modal remote compute
MODAL_TOKEN_ID=$MODAL_TOKEN_ID
MODAL_TOKEN_SECRET=$MODAL_TOKEN_SECRET
```

## Starting the App

```bash
chmod +x start.sh
./start.sh
```

On first run, `start.sh` automatically installs:
- Python packages and dependencies
- Node.js packages
- Gemini CLI
- Scientific skills

App opens at **http://localhost:3000**. Stop with `Ctrl+C`.

## Project Structure

```
k-dense-byok/
├── start.sh              # One-command launcher
├── server.py             # FastAPI backend
├── kady_agent/
│   ├── .env              # API keys
│   ├── agent.py          # Main Kady agent definition
│   └── tools/            # Web search, delegation, file ops
├── web/                  # Next.js frontend
└── sandbox/              # Local file workspace (auto-created)
```

## Key Files to Understand

### `kady_agent/agent.py` — Main Agent

This defines Kady's behavior, system prompt, and tool access. Modify this to change how Kady responds, which tools it uses, or to add custom instructions.

```python
# kady_agent/agent.py (illustrative structure)
from anthropic import Anthropic

client = Anthropic()

# Kady decides: answer directly OR delegate to an expert
# Experts have access to 170+ scientific skills
```

### `kady_agent/tools/` — Tool Definitions

Each file in `tools/` adds a capability to Kady:
- Web search (via Parallel API)
- Expert delegation (fires up a Gemini CLI subprocess)
- File read/write in the sandbox
- Scientific database queries

## Adding a Custom Tool (TypeScript/Backend Pattern)

If extending the backend with a new tool:

```typescript
// web/src/lib/api.ts — example frontend API call pattern
const response = await fetch('http://localhost:8000/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    message: userMessage,
    session_id: sessionId,
    model: selectedModel,       // e.g. "openrouter/anthropic/claude-opus-4-5"
    files: attachedFiles,
  }),
});

const reader = response.body?.getReader();
// Streams back tokens as SSE
```

```typescript
// web/src/components/Chat.tsx — reading streamed response
async function streamResponse(reader: ReadableStreamDefaultReader) {
  const decoder = new TextDecoder();
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const chunk = decoder.decode(value);
    appendToMessage(chunk);
  }
}
```

## Selecting AI Models

The UI dropdown supports 40+ models. You can also set the default in `.env`:

```env
# Examples of valid DEFAULT_AGENT_MODEL values
DEFAULT_AGENT_MODEL=openrouter/google/gemini-3.1-pro-preview
DEFAULT_AGENT_MODEL=openrouter/anthropic/claude-opus-4-5
DEFAULT_AGENT_MODEL=openrouter/openai/gpt-4o
DEFAULT_AGENT_MODEL=openrouter/x-ai/grok-3
DEFAULT_AGENT_MODEL=openrouter/qwen/qwen-2.5-72b-instruct
```

> Note: The dropdown model only controls Kady. Expert/coding delegation always uses a Gemini model via OpenRouter.

## Working with Files

Files go in and come from the `sandbox/` directory. Users can:
- Upload files via the UI
- Ask Kady to create/modify files
- Preview almost any file type in the side panel

```python
# Backend file tool pattern (server.py / tools)
import os

SANDBOX_DIR = os.path.join(os.path.dirname(__file__), "sandbox")

def read_file(filename: str) -> str:
    path = os.path.join(SANDBOX_DIR, filename)
    with open(path, "r") as f:
        return f.read()

def write_file(filename: str, content: str) -> None:
    path = os.path.join(SANDBOX_DIR, filename)
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w") as f:
        f.write(content)
```

## Enabling Web Search

Set `PARALLEL_API_KEY` in `kady_agent/.env`. Kady will automatically use web search when it determines live information is needed — no code changes required.

## Enabling Remote Compute (Modal)

```env
MODAL_TOKEN_ID=$MODAL_TOKEN_ID
MODAL_TOKEN_SECRET=$MODAL_TOKEN_SECRET
```

Once set, Kady's experts can offload heavy computation (large dataset processing, ML training, etc.) to Modal cloud hardware instead of running locally.

## MCP (Model Context Protocol) Configuration

MCP can be configured directly in the code today. UI-based MCP config is on the roadmap. To add an MCP server manually, locate the agent configuration in `kady_agent/agent.py` and add your MCP server definition there.

## Common Patterns

### Ask Kady to use a specific scientific skill

```
"Analyze this genomics dataset using bioinformatics tools"
"Run a financial time-series analysis on this CSV"
"Search PubMed for recent papers on CRISPR delivery mechanisms"
```

### Ask Kady to search + synthesize

```
"Search the web for the latest benchmarks on protein folding models and summarize the findings"
```

### Ask Kady to write and run code

```
"Write a Python script to parse this FASTA file and plot GC content, then run it"
```

### File workflow

```
"I've uploaded data.csv — clean it, run a regression, and save the results as results.csv"
```

## Troubleshooting

**App won't start / port already in use**
```bash
# Kill processes on ports 3000, 4000, 8000
lsof -ti:3000,4000,8000 | xargs kill -9
./start.sh
```

**`OPENROUTER_API_KEY` not found error**
- Confirm `kady_agent/.env` exists (not `.env` in the root)
- Check for typos — no quotes needed around values

**Gemini CLI / expert tasks failing**
- Expert tasks always route through OpenRouter using a Gemini model
- Verify your OpenRouter account has credits and Gemini model access
- Check `GOOGLE_GEMINI_BASE_URL=http://localhost:4000` and `GEMINI_API_KEY=sk-litellm-local` are unchanged

**Scientific skills not downloading**
- First run requires internet access; subsequent runs are cached
- Re-run `./start.sh` — it will retry failed downloads

**Web search not working**
- Confirm `PARALLEL_API_KEY` is set in `kady_agent/.env`
- Verify the key is active at [parallel.ai](https://parallel.ai/)

**Slow first startup**
- Normal — `start.sh` installs Python packages, Node modules, Gemini CLI, and downloads skills
- Subsequent starts are significantly faster

**Windows / WSL issues**
- Run everything inside WSL, not PowerShell/CMD
- File paths must use Linux-style `/` separators inside WSL

## Staying Updated

```bash
git pull origin main
./start.sh   # picks up any new dependencies automatically
```

Star the repo and follow [@k_dense_ai](https://x.com/k_dense_ai) for release notes. Planned additions include Ollama local model support, Claude Code as delegation backend, AutoResearch integration, and improved PDF parsing.
```
