```markdown
---
name: aris-autonomous-ml-research
description: Autonomous ML research workflows using ARIS (Auto-Research-In-Sleep) — Markdown-only skills for cross-model paper review loops, idea discovery, experiment automation, and rebuttal writing with Claude Code, Codex, or any LLM agent.
triggers:
  - "set up ARIS for autonomous research"
  - "run research pipeline while I sleep"
  - "automate ML paper writing with Claude Code"
  - "cross-model paper review loop"
  - "generate research ideas automatically"
  - "write rebuttal for paper reviews"
  - "run experiment pipeline with ARIS"
  - "install ARIS skills for Claude Code"
---

# ARIS — Autonomous ML Research In Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS (Auto-Research-In-Sleep) is a **zero-dependency, Markdown-only** research automation system. It gives LLM coding agents (Claude Code, Codex CLI, Cursor, Trae, Antigravity, Windsurf) a complete pipeline: idea discovery → experiment design → code execution → paper writing → rebuttal. The "skill" is plain `SKILL.md` files — no framework, no lock-in.

---

## What ARIS Does

| Workflow | Command | What happens |
|----------|---------|-------------|
| **0 — Full pipeline** | `/research-pipeline` | Idea → experiments → paper, end-to-end |
| **1 — Idea discovery** | `/idea-discovery` | Literature survey → gap analysis → ranked ideas |
| **1.5 — Experiment bridge** | `/experiment-bridge` | GPT cross-review → GPU run → results |
| **2 — Paper writing** | `/paper-writing` | Results → full LaTeX draft with citations |
| **3 — Paper review** | `/paper-review` | Cross-model adversarial critique loop |
| **4 — Rebuttal** | `/rebuttal` | Reviews → strategy → draft → safety check |
| **Slides** | `/paper-slides` | Paper → Beamer PDF + PPTX + speaker notes |
| **Poster** | `/paper-poster` | Paper → A0/A1 PDF + PPTX + SVG |

**Core design principle:** Claude Code executes (fast, fluid); an external model (GPT, Kimi, GLM, DeepSeek, Gemini) reviews (deliberate, rigorous). Cross-model critique breaks self-play blind spots.

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### 2. Install skills into Claude Code

Copy skill files to Claude Code's custom commands directory:

```bash
# Claude Code skills directory (macOS/Linux)
SKILLS_DIR="$HOME/.claude/commands"
mkdir -p "$SKILLS_DIR"

# Install core skills
cp skills/research-pipeline/SKILL.md "$SKILLS_DIR/research-pipeline.md"
cp skills/idea-discovery/SKILL.md "$SKILLS_DIR/idea-discovery.md"
cp skills/experiment-bridge/SKILL.md "$SKILLS_DIR/experiment-bridge.md"
cp skills/paper-writing/SKILL.md "$SKILLS_DIR/paper-writing.md"
cp skills/paper-review/SKILL.md "$SKILLS_DIR/paper-review.md"
cp skills/rebuttal/SKILL.md "$SKILLS_DIR/rebuttal.md"
cp skills/paper-slides/SKILL.md "$SKILLS_DIR/paper-slides.md"
cp skills/paper-poster/SKILL.md "$SKILLS_DIR/paper-poster.md"
```

### 3. Install the Codex MCP server (reviewer model)

The reviewer model is bridged via the `llm-chat` MCP server:

```bash
cd mcp-servers/llm-chat
pip install -r requirements.txt
```

Configure in `~/.claude/mcp_config.json`:

```json
{
  "mcpServers": {
    "codex": {
      "command": "python",
      "args": ["path/to/Auto-claude-code-research-in-sleep/mcp-servers/llm-chat/server.py"],
      "env": {
        "OPENAI_API_KEY": "$OPENAI_API_KEY",
        "OPENAI_MODEL": "gpt-4o"
      }
    }
  }
}
```

### 4. Set environment variables

```bash
# Required for OpenAI reviewer (or use alternative models below)
export OPENAI_API_KEY="your-key-here"

# Optional: Anthropic key if running Claude Code programmatically
export ANTHROPIC_API_KEY="your-key-here"
```

---

## Quick Start

### Full autonomous pipeline

```
/research-pipeline "factorized gap in discrete diffusion language models"
```

### Targeted improvement (paper + repo)

```
/research-pipeline "improve attention efficiency" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

### Just idea discovery

```
/idea-discovery "self-supervised learning for low-resource NLP"
```

### Run experiments only

```
/experiment-bridge "experiments/" — gpu: true, code review: true
```

### Write paper from results

```
/paper-writing "results/" — venue: NeurIPS, template: neurips
```

### Review your own paper

```
/paper-review "paper/" — rounds: 3, target score: 7
```

### Rebuttal after reviews come in

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

### Slides and poster

```
/paper-slides "paper/"
/paper-poster "paper/" — format: A0
```

---

## Workflow Details

### Workflow 0: Full Research Pipeline

```
/research-pipeline "<topic>" [options]
```

**Options:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `ref paper` | — | ArXiv URL to read, find weaknesses, improve |
| `base repo` | — | GitHub URL to clone as codebase |
| `venue` | `NeurIPS` | Target conference |
| `compact` | `false` | Generate lean summary files for short-context models |

**What ARIS does automatically:**
1. Searches literature (Semantic Scholar, ArXiv, DBLP)
2. Identifies research gaps
3. Generates and ranks ideas
4. Designs experiments
5. Writes and executes code (with cross-model code review)
6. Runs on GPU/CPU
7. Parses results
8. Writes LaTeX paper
9. Self-reviews and iterates

---

### Workflow 1: Idea Discovery

```
/idea-discovery "<research direction>"
```

Produces `ideas.md` with ranked ideas, each containing:
- Problem statement
- Proposed solution
- Why it's novel (gap analysis)
- Experiment sketch
- Estimated difficulty

---

### Workflow 1.5: Experiment Bridge

```
/experiment-bridge "<experiment dir>" [options]
```

**Options:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `gpu` | `false` | Submit to GPU cluster |
| `code review` | `true` | Cross-model GPT review before execution |
| `wandb` | `false` | Enable W&B logging via `wandb.Api()` |
| `max retries` | `3` | Auto-retry on failure |

**Code review integration** — before any GPU job is submitted, the reviewer model inspects:
- Correctness of training loop
- Data pipeline bugs
- Hyperparameter sanity
- Resource estimates

---

### Workflow 3: Paper Review Loop

```
/paper-review "<paper dir>" [options]
```

**Options:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `rounds` | `3` | Max review-revise cycles |
| `target score` | `7` | Stop when score ≥ this (out of 10) |
| `reviewer model` | `gpt-4o` | Model used for critique |

**Review loop:**
```
Claude writes/revises → GPT reviews → score + weaknesses → Claude revises → repeat
```

Each round produces:
- `review_round_N.md` — detailed critique
- `score_round_N.txt` — numeric score + justification
- Revised paper section

The score progression chart (`auto_review_score_curve.png`) is auto-generated.

---

### Workflow 4: Rebuttal

```
/rebuttal "<paper dir> + <reviews dir>" — venue: ICML, character limit: 5000
```

**Options:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | ICML/NeurIPS/ICLR/CVPR/ACL/AAAI/ACM |
| `character limit` | required | Hard limit for rebuttal text |
| `quick mode` | `false` | Stop after Phase 3 (strategy only, no draft) |
| `auto experiment` | `false` | Auto-run supplementary experiments |
| `max stress test rounds` | `1` | GPT stress-test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up rounds |

**Pipeline phases:**
```
Phase 0: Parse reviews
Phase 1: Atomize concerns (one claim per line)
Phase 2: Categorize (fatal / major / minor / question)
Phase 3: Strategy (what to concede, what to defend)
Phase 4: Draft rebuttal
Phase 5: Safety check (3 gates)
Phase 6: GPT stress-test
Phase 7: Finalize → PASTE_READY.txt + REBUTTAL_DRAFT_rich.md
Phase 8: Per-reviewer follow-up rounds
```

**Three safety gates (rebuttal won't finalize if any fails):**
- 🔒 No fabrication — every claim maps to paper/review/confirmed result
- 🔒 No overpromise — every promise is user-approved
- 🔒 Full coverage — every reviewer concern is addressed

**Outputs:**
- `PASTE_READY.txt` — exact char count, paste directly to venue system
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

---

## Alternative Model Combinations

ARIS works with **any OpenAI-compatible API**. No Claude or OpenAI required.

### Configure custom reviewer model

Edit `mcp-servers/llm-chat/server.py` config or set env vars:

```bash
# MiniMax as reviewer
export LLM_CHAT_BASE_URL="https://api.minimax.chat/v1"
export LLM_CHAT_API_KEY="$MINIMAX_API_KEY"
export LLM_CHAT_MODEL="minimax-m2"

# Kimi as reviewer
export LLM_CHAT_BASE_URL="https://api.moonshot.cn/v1"
export LLM_CHAT_API_KEY="$MOONSHOT_API_KEY"
export LLM_CHAT_MODEL="moonshot-v1-128k"

# DeepSeek as reviewer
export LLM_CHAT_BASE_URL="https://api.deepseek.com/v1"
export LLM_CHAT_API_KEY="$DEEPSEEK_API_KEY"
export LLM_CHAT_MODEL="deepseek-chat"

# GLM as reviewer
export LLM_CHAT_BASE_URL="https://open.bigmodel.cn/api/paas/v4"
export LLM_CHAT_API_KEY="$GLM_API_KEY"
export LLM_CHAT_MODEL="glm-4"
```

### MCP config for custom model

```json
{
  "mcpServers": {
    "llm-reviewer": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_CHAT_BASE_URL": "https://api.deepseek.com/v1",
        "LLM_CHAT_API_KEY": "$DEEPSEEK_API_KEY",
        "LLM_CHAT_MODEL": "deepseek-chat"
      }
    }
  }
}
```

### Free tier via ModelScope

```bash
# Zero cost option
export LLM_CHAT_BASE_URL="https://api-inference.modelscope.cn/v1"
export LLM_CHAT_API_KEY="$MODELSCOPE_API_KEY"
export LLM_CHAT_MODEL="Qwen/Qwen2.5-72B-Instruct"
```

See `docs/MODELSCOPE_GUIDE.md` for full setup.

---

## Using with Other Agents

### Codex CLI

Skills are also available in `skills/skills-codex/`:

```bash
# Install Codex-native skills
cp skills/skills-codex/*.md ~/.codex/commands/
```

### Cursor

```bash
# Skills work as Cursor custom commands
cp skills/*/SKILL.md ~/.cursor/commands/
```

See `docs/CURSOR_ADAPTATION.md` for Cursor-specific MCP config.

### Trae (ByteDance IDE)

See `docs/TRAE_ARIS_RUNBOOK_EN.md` for full setup guide.

### Antigravity (Google)

```bash
# Antigravity supports SKILL.md natively
cp skills/*/SKILL.md ~/.antigravity/skills/
```

See `docs/ANTIGRAVITY_ADAPTATION.md`.

---

## Paper Templates

ARIS ships with venue-specific LaTeX templates in `templates/`:

```
templates/
  neurips/      # NeurIPS
  icml/         # ICML
  iclr/         # ICLR
  cvpr/         # CVPR
  acl/          # ACL
  aaai/         # AAAI
  acm/          # ACM MM / ACM general
```

Use a template explicitly:

```
/paper-writing "results/" — venue: CVPR, template: cvpr
```

---

## Input Templates

Every workflow has a fill-in-the-blank input template in `templates/inputs/`:

```bash
# Copy and edit the template for your workflow
cp templates/inputs/research-pipeline.md my-research-input.md
# Edit my-research-input.md, then:
# /research-pipeline — input: my-research-input.md
```

---

## Compact Mode (Short-Context Models)

For models with limited context windows or after long sessions:

```
/research-pipeline "topic" — compact: true
```

Generates lean `*_compact.md` summary files at each checkpoint. Also used for session recovery after interruption:

```
/research-pipeline — resume: checkpoints/step3_compact.md
```

---

## Programmatic Usage (Python)

Although ARIS is Markdown-first, you can script interactions with the MCP server directly:

```python
import asyncio
import json
import subprocess

async def call_reviewer(prompt: str, model: str = "gpt-4o") -> str:
    """Call the llm-chat MCP server programmatically."""
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {
            "name": "llm_chat",
            "arguments": {
                "message": prompt,
                "model": model
            }
        },
        "id": 1
    }
    # MCP stdio transport
    proc = await asyncio.create_subprocess_exec(
        "python", "mcp-servers/llm-chat/server.py",
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
    )
    stdout, _ = await proc.communicate(json.dumps(payload).encode())
    return json.loads(stdout)["result"]["content"][0]["text"]

# Example: run a review round
async def main():
    paper_text = open("paper/draft.md").read()
    review = await call_reviewer(
        f"Review this ML paper section critically:\n\n{paper_text}\n\n"
        f"Score it 1-10 and list the top 3 weaknesses."
    )
    print(review)

asyncio.run(main())
```

---

## Score Curve Visualization

ARIS auto-generates a score progression chart after each review loop:

```python
# scripts/plot_score_curve.py (included in repo)
import matplotlib.pyplot as plt
import json

def plot_score_curve(scores_file: str = "review_scores.json"):
    with open(scores_file) as f:
        scores = json.load(f)  # {"rounds": [1,2,3], "scores": [4.5, 6.0, 7.2]}
    
    plt.figure(figsize=(8, 4))
    plt.plot(scores["rounds"], scores["scores"], marker="o", linewidth=2)
    plt.axhline(y=7, color="green", linestyle="--", label="Accept threshold")
    plt.xlabel("Review Round")
    plt.ylabel("Score / 10")
    plt.title("ARIS Auto-Review Score Progression")
    plt.legend()
    plt.tight_layout()
    plt.savefig("docs/auto_review_score_curve.png", dpi=150)
    print("Saved score curve.")

plot_score_curve()
```

Run manually:

```bash
python scripts/plot_score_curve.py
```

---

## Utility Skills

### Formula derivation

```
/formula-derivation "derive the ELBO for discrete diffusion"
```

Produces step-by-step derivation with LaTeX, verified for consistency.

### Citation verification

```
/paper-writing "results/" — verify citations: true
```

Each citation goes through: DBLP → CrossRef → `[VERIFY]` flag if not found. Prevents hallucinated references.

### Training check

Integrated into Workflow 1.5 — validates training loop before GPU submission:

```
/experiment-bridge "experiments/" — training check: true
```

### Ablation planner

```
/experiment-bridge "experiments/" — ablation: true
```

Auto-generates ablation study design based on your method's components.

---

## Troubleshooting

### MCP server not connecting

```bash
# Test the llm-chat server directly
echo '{"jsonrpc":"2.0","method":"tools/list","params":{},"id":1}' | \
  python mcp-servers/llm-chat/server.py

# Should return list of available tools
```

### Skill not found in Claude Code

```bash
# Verify skill is in the right directory
ls ~/.claude/commands/
# Should list research-pipeline.md, paper-review.md, etc.

# Reload Claude Code after copying skills
```

### Review loop produces no improvement

- Switch reviewer model: GPT-4o → DeepSeek → Kimi (different blind spots)
- Increase rounds: `— rounds: 5`
- Add explicit weakness focus: `/paper-review "paper/" — focus: methodology,experiments`

### LaTeX compilation fails

```bash
# ARIS uses pdflatex by default
which pdflatex  # Must be installed

# Install TeX Live (Ubuntu)
sudo apt-get install texlive-full

# macOS
brew install --cask mactex
```

### Rebuttal exceeds character limit

- The `PASTE_READY.txt` output is hard-trimmed to your specified limit
- Use `quick mode: true` first to see the strategy before drafting
- Increase `max stress test rounds` to get tighter drafts

### Context window exhausted mid-pipeline

```
/research-pipeline — resume: checkpoints/latest_compact.md
```

Enable compact mode from the start for long research sessions:

```
/research-pipeline "topic" — compact: true
```

---

## Project Structure

```
Auto-claude-code-research-in-sleep/
├── skills/
│   ├── research-pipeline/SKILL.md     # Workflow 0
│   ├── idea-discovery/SKILL.md        # Workflow 1
│   ├── experiment-bridge/SKILL.md     # Workflow 1.5
│   ├── paper-writing/SKILL.md         # Workflow 2
│   ├── paper-review/SKILL.md          # Workflow 3
│   ├── rebuttal/SKILL.md              # Workflow 4
│   ├── paper-slides/SKILL.md
│   ├── paper-poster/SKILL.md
│   ├── formula-derivation/SKILL.md
│   ├── grant-proposal/SKILL.md
│   ├── paper-illustration/SKILL.md
│   └── skills-codex/                  # Codex CLI versions
├── mcp-servers/
│   └── llm-chat/                      # Reviewer model bridge
│       ├── server.py
│       └── requirements.txt
├── templates/
│   ├── neurips/ icml/ iclr/ cvpr/ acl/ aaai/ acm/
│   └── inputs/                        # Fill-in-the-blank input templates
├── docs/
│   ├── CURSOR_ADAPTATION.md
│   ├── TRAE_ARIS_RUNBOOK_EN.md
│   ├── ANTIGRAVITY_ADAPTATION.md
│   ├── MODELSCOPE_GUIDE.md
│   ├── CODEX_GEMINI_REVIEW_GUIDE.md
│   └── MiniMax-GLM-Configuration.md
└── scripts/
    └── plot_score_curve.py
```

---

## Key Design Principles

1. **Zero dependencies** — every skill is a plain `.md` file. No pip install, no Docker, no daemon.
2. **No lock-in** — swap the executor (Claude Code → Codex → Cursor) or reviewer (GPT → Kimi → DeepSeek) freely.
3. **Cross-model adversarial review** — the biggest quality gain is going from 1 model to 2. ARIS uses 2-player review by design.
4. **Checkpoint-first** — every workflow saves progress. Interruptions are recoverable.
5. **Safety gates in rebuttal** — ARIS won't let you submit fabricated claims or broken promises.
```
