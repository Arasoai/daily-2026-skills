```markdown
---
name: openspace-self-evolving-agents
description: Skill for using OpenSpace to make AI agents smarter, lower-cost, and self-evolving through skill sharing and collective intelligence.
triggers:
  - add OpenSpace to my agent
  - make my agent self-evolving
  - reduce agent token costs
  - share skills between agents
  - plug OpenSpace into Claude Code
  - set up skill evolution for my AI agent
  - use OpenSpace MCP server
  - evolve agent skills automatically
---

# OpenSpace: Self-Evolving Agent Skills

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

OpenSpace is a self-evolving engine that plugs into any MCP-compatible AI agent (Claude Code, Codex, OpenClaw, nanobot, Cursor, etc.) and gives it three superpowers: **self-evolving skills** (auto-fix, auto-improve, auto-learn), **collective agent intelligence** (shared skill community), and **token efficiency** (46% fewer tokens, 4.2× better economic output on real-world tasks).

---

## Installation

```bash
git clone https://github.com/HKUDS/OpenSpace.git && cd OpenSpace
pip install -e .
openspace-mcp --help   # verify installation
```

Node.js ≥ 20 is required only for the local dashboard frontend.

---

## Two Usage Paths

### Path A — Plug Into Your Agent (MCP)

Add OpenSpace as an MCP server in your agent's config file:

```json
{
  "mcpServers": {
    "openspace": {
      "command": "openspace-mcp",
      "toolTimeout": 600,
      "env": {
        "OPENSPACE_HOST_SKILL_DIRS": "/path/to/your/agent/skills",
        "OPENSPACE_WORKSPACE": "/path/to/OpenSpace",
        "OPENSPACE_API_KEY": "$OPENSPACE_API_KEY"
      }
    }
  }
}
```

Then copy the two bootstrap host skills into your agent's skills directory:

```bash
cp -r OpenSpace/openspace/host_skills/delegate-task/ /path/to/your/agent/skills/
cp -r OpenSpace/openspace/host_skills/skill-discovery/ /path/to/your/agent/skills/
```

These two skills teach the agent **when and how** to use OpenSpace — no additional prompting needed.

- `delegate-task` — delegate complex tasks to OpenSpace for execution with evolved skills
- `skill-discovery` — let the agent search for and download community skills

> Credentials (LLM API key, model) are auto-detected from the agent's own config. `OPENSPACE_API_KEY` is optional — local features work without it.

### Path B — Use OpenSpace Directly as a Co-Worker

Create a `.env` file (see `openspace/.env.example`):

```bash
# openspace/.env
ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY           # or OPENAI_API_KEY, etc.
OPENSPACE_API_KEY=$OPENSPACE_API_KEY           # optional, for cloud community
```

Then run interactively or with a query:

```bash
# Interactive REPL
openspace

# One-shot task
openspace --model "anthropic/claude-sonnet-4-5" \
  --query "Create a monitoring dashboard for my Docker containers"
```

---

## Python API

```python
import asyncio
from openspace import OpenSpace

async def main():
    async with OpenSpace() as cs:
        # Execute a task — skills evolve automatically
        result = await cs.execute(
            "Analyze GitHub trending repos and create a report"
        )
        print(result["response"])

        # Inspect which skills evolved during this session
        for skill in result.get("evolved_skills", []):
            print(f"  Evolved: {skill['name']} (origin: {skill['origin']})")

asyncio.run(main())
```

### Execute with a specific model

```python
import asyncio
from openspace import OpenSpace

async def run_task():
    async with OpenSpace(model="openai/gpt-4o") as cs:
        result = await cs.execute(
            "Build a payroll calculator from this union contract PDF"
        )
        print(result["response"])
        print(f"Tokens used: {result.get('token_usage')}")

asyncio.run(run_task())
```

---

## CLI Reference

| Command | Description |
|---|---|
| `openspace` | Interactive agent REPL |
| `openspace --query "..."` | Execute a single task |
| `openspace --model "provider/model"` | Specify LLM model |
| `openspace-mcp` | Start the MCP server (for agent integration) |
| `openspace-dashboard --port 7788` | Start local dashboard API backend |
| `openspace-download-skill <skill_id>` | Download a skill from the cloud community |
| `openspace-upload-skill /path/to/skill/` | Upload a local skill to the cloud community |

---

## Cloud Skill Community

Register at [open-space.cloud](https://open-space.cloud) to get an API key, then set it in env:

```bash
export OPENSPACE_API_KEY="$OPENSPACE_API_KEY"
```

Or add to `.env`:

```
OPENSPACE_API_KEY=$OPENSPACE_API_KEY
```

### Download a community skill

```bash
openspace-download-skill skill_abc123
```

### Upload an evolved skill

```bash
openspace-upload-skill ./openspace/skills/my-evolved-skill/
```

Skills can be set to **public**, **private**, or **team-only** access.

---

## Local Dashboard

Visualize skill evolution lineage, browse versions, compare diffs, and inspect execution sessions.

```bash
# Terminal 1 — backend API
openspace-dashboard --port 7788

# Terminal 2 — frontend dev server
cd frontend
npm install      # only once
npm run dev      # opens at http://localhost:5173
```

Dashboard panels:
- **Skill Classes** — browse, search, and sort all local skills
- **Cloud** — discover community skill records
- **Version Lineage** — visual evolution graph per skill
- **Workflow Sessions** — execution history and token metrics

---

## Writing Custom Skills

Skills live as directories under `openspace/skills/`. Each skill directory contains:

```
my-skill/
  SKILL.md          # skill description and usage instructions
  skill.py          # implementation (Python)
  requirements.txt  # optional dependencies
```

See `openspace/skills/README.md` for the full authoring guide.

Example minimal skill:

```python
# openspace/skills/fetch-url/skill.py

import httpx

async def fetch_url(url: str) -> dict:
    """Fetch content from a URL and return text and status."""
    async with httpx.AsyncClient(timeout=30) as client:
        response = await client.get(url)
        return {
            "status": response.status_code,
            "text": response.text[:4000],
            "url": url,
        }
```

---

## Environment Variables Reference

| Variable | Required | Description |
|---|---|---|
| `OPENSPACE_HOST_SKILL_DIRS` | For Path A | Path to agent's skills directory |
| `OPENSPACE_WORKSPACE` | For Path A | Path to OpenSpace repo root |
| `OPENSPACE_API_KEY` | Optional | Cloud community access key |
| `ANTHROPIC_API_KEY` | If using Claude | Anthropic API key |
| `OPENAI_API_KEY` | If using OpenAI | OpenAI API key |

All env vars can be set in `openspace/.env` (copy from `openspace/.env.example`).

---

## Common Patterns

### Pattern: Delegate a complex multi-step task

```python
async with OpenSpace() as cs:
    result = await cs.execute(
        "Prepare a tax return summary from the 15 PDFs in ./documents/tax/"
    )
    # OpenSpace selects or evolves the right skills automatically
    print(result["response"])
```

### Pattern: Reuse evolved skills across agents

```bash
# Agent A evolves a skill, upload it
openspace-upload-skill ./openspace/skills/tax-return-summarizer/

# Agent B downloads and uses it immediately
openspace-download-skill skill_tax_return_abc123
```

### Pattern: Monitor skill quality

Skills auto-track: performance, error rates, and execution success. View in the dashboard or query programmatically:

```python
async with OpenSpace() as cs:
    result = await cs.execute("Show skill performance metrics for the last 7 days")
    print(result["response"])
```

---

## Per-Agent MCP Configuration

### Claude Code (`claude_desktop_config.json` or `.claude/mcp.json`)

```json
{
  "mcpServers": {
    "openspace": {
      "command": "openspace-mcp",
      "toolTimeout": 600,
      "env": {
        "OPENSPACE_HOST_SKILL_DIRS": "/Users/me/.claude/skills",
        "OPENSPACE_WORKSPACE": "/Users/me/OpenSpace",
        "OPENSPACE_API_KEY": "$OPENSPACE_API_KEY"
      }
    }
  }
}
```

### Codex / OpenClaw / nanobot

Follow the same MCP config pattern — consult `openspace/host_skills/README.md` for agent-specific paths and any extra flags.

---

## Troubleshooting

**`openspace-mcp` not found after install**
```bash
pip install -e .   # re-run from repo root
which openspace-mcp
```

**Skills not evolving / auto-fix not triggering**
- Ensure both `delegate-task` and `skill-discovery` host skills are copied to your agent's skills directory.
- Check that `OPENSPACE_WORKSPACE` points to the repo root (where `openspace/skills/` lives).

**Cloud upload/download fails**
- Verify `OPENSPACE_API_KEY` is set and valid (register at [open-space.cloud](https://open-space.cloud)).
- Check network connectivity to the cloud endpoint.

**Dashboard frontend won't start**
- Node.js ≥ 20 is required: `node --version`
- Run `npm install` inside the `frontend/` directory before `npm run dev`.

**LLM credentials not picked up**
- OpenSpace auto-detects credentials from the host agent's environment. If running standalone (Path B), set keys in `openspace/.env`.

---

## Related Projects

- [ClawWork](https://github.com/HKUDS/ClawWork) — evaluation protocol used in GDPVal benchmark
- [nanobot](https://github.com/HKUDS/nanobot) — lightweight agent compatible with OpenSpace
- [OpenClaw](https://github.com/openclaw/openclaw) — agent framework compatible with OpenSpace
- [GDPVal dataset](https://huggingface.co/datasets/openai/gdpval) — 220 real-world professional tasks benchmark
- Community & skill browser: [open-space.cloud](https://open-space.cloud)
```
