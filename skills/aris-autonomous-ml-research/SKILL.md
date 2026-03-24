```markdown
---
name: aris-autonomous-ml-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only skill system for autonomous ML research workflows using cross-model review loops, idea discovery, experiment automation, and paper writing with Claude Code and any LLM agent.
triggers:
  - set up ARIS for autonomous research
  - run research pipeline while I sleep
  - automate ML paper writing with Claude Code
  - cross-model review loop for my paper
  - use ARIS to improve my research paper
  - run experiment automation with ARIS
  - set up auto paper review with Claude Code and GPT
  - install ARIS research skills
---

# ARIS — Autonomous ML Research In Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS (Auto-Research-In-Sleep) is a **zero-dependency, Markdown-only** autonomous ML research system. It orchestrates cross-model collaboration where one LLM (e.g. Claude Code) drives execution while another (e.g. GPT-5.4 via Codex MCP) acts as adversarial reviewer — breaking self-play blind spots that single-model loops fall into.

The entire system is plain `.md` files. No framework, no database, no Docker. Every skill is a `SKILL.md` readable by any agent: Claude Code, Codex CLI, Cursor, Trae, Antigravity, Windsurf, or your own.

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### 2. Install skills into Claude Code

Copy skills into your project's `.claude/` directory:

```bash
# Copy all core skills
cp -r skills/ /your-project/.claude/skills/

# Or symlink for live updates
ln -s $(pwd)/skills /your-project/.claude/skills
```

For Claude Code global install (available in all projects):

```bash
cp -r skills/ ~/.claude/skills/
```

### 3. Configure the Codex MCP reviewer (cross-model setup)

ARIS uses a second LLM as adversarial reviewer via MCP. Default: Claude Code + Codex (GPT-5.4).

Create or update `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "codex": {
      "command": "npx",
      "args": ["-y", "@openai/codex-mcp"],
      "env": {
        "OPENAI_API_KEY": "$OPENAI_API_KEY"
      }
    }
  }
}
```

For alternative models (no OpenAI required), use the bundled `llm-chat` MCP server:

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "$LLM_API_KEY",
        "LLM_BASE_URL": "$LLM_BASE_URL",
        "LLM_MODEL": "glm-5"
      }
    }
  }
}
```

### 4. Set environment variables

```bash
# For Claude Code (primary executor)
export ANTHROPIC_API_KEY=your_key_here

# For Codex reviewer (optional — use llm-chat for alternatives)
export OPENAI_API_KEY=your_key_here

# For alternative reviewer models (DeepSeek, Kimi, GLM, MiniMax, etc.)
export LLM_API_KEY=your_key_here
export LLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4/  # example: GLM
export LLM_MODEL=glm-5
```

---

## Core Workflows & Commands

All ARIS commands are invoked as Claude Code slash commands after skills are installed.

### Workflow 0: Full Research Pipeline (start here)

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

**Targeted mode** — give it an existing paper + codebase to improve:

```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

Mix and match:
- `ref paper` only → "what can be improved?"
- `base repo` only → "what can I build with this code?"
- Both → "improve this paper using this code"

### Workflow 1: Idea Discovery

```
/idea-discovery "discrete diffusion language models"
```

Searches literature → identifies gaps → generates 5 novel hypotheses → cross-model review → ranks by novelty + feasibility.

### Workflow 1.5: Experiment Bridge

```
/experiment-bridge "run ablation on latent bottleneck width" — base repo: https://github.com/org/repo
```

Parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `code review` | `true` | GPT-5.4 reviews code before GPU deployment |
| `wandb` | `false` | Enable W&B logging |
| `gpu` | auto | Target GPU spec |

### Workflow 2: Literature Review

```
/literature-review "score-based generative models for text"
```

Fetches via DBLP → CrossRef → marks unverified as `[VERIFY]` (anti-hallucination).

### Workflow 3: Paper Writing

```
/paper-write "results/" — venue: NeurIPS
```

Supported venues: ICML, NeurIPS, ICLR, CVPR, ACL, AAAI, ACM MM.

### Workflow 4: Rebuttal

```
/rebuttal "paper/ + reviews/" — venue: ICML, character limit: 5000
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | Target conference |
| `character limit` | required | Hard char limit for paste |
| `quick mode` | `false` | Stop after strategy (no draft) |
| `auto experiment` | `false` | Auto-run supplementary experiments |
| `max stress test rounds` | `1` | GPT-5.4 stress test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up limit |

Outputs: `PASTE_READY.txt` (exact char count) + `REBUTTAL_DRAFT_rich.md`.

Three safety gates — rebuttal will NOT finalize if any fails:
- 🔒 No fabrication — every claim maps to paper/review/confirmed result
- 🔒 No overpromise — every promise is user-approved
- 🔒 Full coverage — every reviewer concern is tracked

### Presentation outputs

```bash
/paper-slides "paper/"   # → Beamer PDF + PPTX + speaker notes + Q&A prep
/paper-poster "paper/"   # → A0/A1 poster PDF + editable PPTX + SVG
```

---

## Alternative Model Combinations

ARIS works with any OpenAI-compatible API. No Claude or OpenAI required.

### MiniMax-M2.7 + GLM-5 (fully free alternative)

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "$MINIMAX_API_KEY",
        "LLM_BASE_URL": "https://api.minimax.chat/v1/",
        "LLM_MODEL": "MiniMax-M2.7"
      }
    }
  }
}
```

Full guide: `docs/MiniMax-GLM-Configuration.md`

### Tested reviewer models

| Model | Provider | Guide |
|-------|----------|-------|
| GPT-5.4 xhigh | OpenAI | Default |
| GLM-5 | Zhipu AI | `llm-chat` MCP |
| MiniMax-M2.7 | MiniMax | `llm-chat` MCP |
| Kimi-K2.5 | Moonshot | `llm-chat` MCP |
| DeepSeek-R2 | DeepSeek | `llm-chat` MCP |
| Gemini 2.5 Pro | Google | `docs/CODEX_GEMINI_REVIEW_GUIDE.md` |

### Free tier via ModelScope

Zero cost, zero lock-in. See `docs/MODELSCOPE_GUIDE.md`.

---

## Project Structure

```
Auto-claude-code-research-in-sleep/
├── skills/                        # Core SKILL.md files (install these)
│   ├── research-pipeline/
│   │   └── SKILL.md
│   ├── idea-discovery/
│   │   └── SKILL.md
│   ├── literature-review/
│   │   └── SKILL.md
│   ├── experiment-bridge/
│   │   └── SKILL.md
│   ├── paper-write/
│   │   └── SKILL.md
│   ├── rebuttal/
│   │   └── SKILL.md
│   ├── paper-slides/
│   │   └── SKILL.md
│   ├── paper-poster/
│   │   └── SKILL.md
│   ├── research-refine/
│   │   └── SKILL.md
│   ├── experiment-plan/
│   │   └── SKILL.md
│   ├── ablation-planner/
│   │   └── SKILL.md
│   ├── training-check/
│   │   └── SKILL.md
│   ├── result-to-claim/
│   │   └── SKILL.md
│   ├── formula-derivation/
│   │   └── SKILL.md
│   └── skills-codex/              # Codex CLI native versions
├── mcp-servers/
│   └── llm-chat/                  # Bring-your-own-model MCP server
│       └── server.py
├── templates/                     # Input templates for every workflow
├── docs/
│   ├── CURSOR_ADAPTATION.md
│   ├── TRAE_ARIS_RUNBOOK_EN.md
│   ├── ANTIGRAVITY_ADAPTATION.md
│   ├── MODELSCOPE_GUIDE.md
│   ├── CODEX_GEMINI_REVIEW_GUIDE.md
│   └── MiniMax-GLM-Configuration.md
└── .mcp.json                      # MCP server configuration
```

---

## Using Templates

ARIS ships with input templates for structured invocation:

```bash
# Copy a template for your workflow
cp templates/research-pipeline-template.md my-research-brief.md

# Edit with your topic, constraints, venue
nano my-research-brief.md

# Pass to Claude Code
/research-pipeline — template: my-research-brief.md
```

---

## Compact Mode (for short-context models)

Generate lean summary files for session recovery or short-context models:

```
/research-pipeline "topic" — compact: true
```

Produces `COMPACT_SUMMARY.md` — a minimal state file for resuming interrupted sessions.

To resume after interruption:

```
/research-pipeline — resume: COMPACT_SUMMARY.md
```

---

## Using the `llm-chat` MCP Server

The bundled MCP server lets any OpenAI-compatible API act as a reviewer.

```python
# mcp-servers/llm-chat/server.py — usage pattern
import os
import openai

client = openai.OpenAI(
    api_key=os.environ["LLM_API_KEY"],
    base_url=os.environ["LLM_BASE_URL"],
)

response = client.chat.completions.create(
    model=os.environ["LLM_MODEL"],
    messages=[
        {"role": "system", "content": "You are an adversarial ML paper reviewer."},
        {"role": "user", "content": review_prompt}
    ]
)
```

Run the server standalone for testing:

```bash
cd mcp-servers/llm-chat
LLM_API_KEY=$YOUR_KEY \
LLM_BASE_URL=https://api.deepseek.com/v1/ \
LLM_MODEL=deepseek-r2 \
python server.py
```

---

## Adding a Custom Skill

Every skill is a plain Markdown file. To create your own:

```bash
mkdir skills/my-custom-skill
cat > skills/my-custom-skill/SKILL.md << 'EOF'
# My Custom Skill

## Trigger
When user asks to [your trigger condition]...

## Steps
1. [Step 1 — what Claude Code does]
2. [Step 2 — cross-model review via MCP]
3. [Step 3 — output]

## Output
- `OUTPUT_FILE.md` — ...
EOF
```

Install it:

```bash
cp -r skills/my-custom-skill ~/.claude/skills/
```

---

## IDE-Specific Setup

### Cursor

See `docs/CURSOR_ADAPTATION.md`. Skills work via Cursor's custom instructions — paste `SKILL.md` content into project rules.

### Trae (ByteDance AI IDE)

See `docs/TRAE_ARIS_RUNBOOK_EN.md`. Full runbook for Trae's agent mode.

### Google Antigravity

See `docs/ANTIGRAVITY_ADAPTATION.md`. Native `SKILL.md` support with dual-model config.

### Codex CLI (OpenAI)

Dedicated skill set in `skills/skills-codex/` — same workflows, Codex-native syntax.

```bash
# Example Codex CLI invocation
codex "run /research-pipeline on factorized gaps in discrete diffusion LMs" \
  --skills skills/skills-codex/
```

---

## Troubleshooting

### MCP server not connecting

```bash
# Verify .mcp.json is in project root
ls -la .mcp.json

# Test MCP server directly
npx @openai/codex-mcp --test

# Check Claude Code MCP status
# In Claude Code: /mcp status
```

### Skills not triggering

```bash
# Verify skills are in the right location
ls ~/.claude/skills/
# or
ls .claude/skills/

# Skills must be named SKILL.md exactly
find . -name "SKILL.md" | head -20
```

### Cross-model review not running

Check that `LLM_API_KEY` / `OPENAI_API_KEY` is set and the MCP server appears in Claude Code's tool list. In Claude Code, run `/mcp` to list available servers.

### W&B logging errors

```bash
pip install wandb
wandb login  # uses $WANDB_API_KEY or interactive
```

ARIS uses `wandb.Api()` for real run tracking — ensure project name matches:

```python
import wandb
api = wandb.Api()
runs = api.runs(f"{os.environ['WANDB_ENTITY']}/{project_name}")
```

### Session interrupted mid-pipeline

```bash
# Resume from checkpoint
/research-pipeline — resume: COMPACT_SUMMARY.md

# Or use research-refine checkpoint
/research-refine — resume: true
```

### Citation hallucination (`[VERIFY]` markers)

ARIS Workflow 2 enforces DBLP → CrossRef verification. Any unverified citation is marked `[VERIFY]` — manually confirm before submission. Do NOT remove markers without verification.

---

## Key Design Principles

1. **Cross-model adversarial review** — one model executes, a different model critiques. 1→2 models gives the biggest blind-spot reduction; more than 2 has diminishing returns.
2. **Speed × Rigor** — Claude Code for fast fluid execution; GPT-5.4 / GLM-5 for deliberate critique.
3. **Zero lock-in** — swap any component. ARIS is a methodology, not a platform.
4. **Markdown-only** — every skill is human-readable, version-controllable, and portable.
5. **Anti-hallucination by design** — citations verified, claims grounded, rebuttal safety-gated.
```
