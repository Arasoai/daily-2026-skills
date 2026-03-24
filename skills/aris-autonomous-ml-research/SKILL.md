```markdown
---
name: aris-autonomous-ml-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only skill system for autonomous ML research workflows using cross-model review loops, idea discovery, experiment automation, and paper writing with Claude Code, Codex, or any LLM agent.
triggers:
  - "set up ARIS for autonomous research"
  - "run research pipeline while I sleep"
  - "automate ML paper writing with Claude Code"
  - "cross-model research review loop"
  - "auto generate research ideas and run experiments"
  - "write my paper autonomously with ARIS"
  - "set up rebuttal pipeline for paper reviews"
  - "install ARIS research skills for Claude Code"
---

# ARIS — Autonomous ML Research In Sleep (⚔️🌙)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous ML research toolkit. It orchestrates cross-model collaboration (e.g., Claude Code executes + GPT-5.4/Codex reviews) through plain `SKILL.md` files — no framework, no lock-in, no daemon. Install once, run research while you sleep.

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### 2. Install Skills into Claude Code

Copy the skills directory into your Claude Code configuration:

```bash
# Install all core skills
cp -r skills/ ~/.claude/skills/

# Or install individual skills
cp skills/research-pipeline/SKILL.md ~/.claude/skills/research-pipeline.md
cp skills/rebuttal/SKILL.md ~/.claude/skills/rebuttal.md
cp skills/experiment-bridge/SKILL.md ~/.claude/skills/experiment-bridge.md
```

### 3. Configure the MCP Reviewer (Codex / OpenAI-compatible)

ARIS uses a second model as an adversarial reviewer via MCP. Set up the `llm-chat` MCP server:

```bash
cd mcp-servers/llm-chat
pip install -r requirements.txt
```

Add to your Claude Code MCP config (`~/.claude/mcp_config.json`):

```json
{
  "mcpServers": {
    "codex-reviewer": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "$OPENAI_API_KEY",
        "LLM_BASE_URL": "https://api.openai.com/v1",
        "LLM_MODEL": "gpt-4o"
      }
    }
  }
}
```

### 4. Alternative: Use Any OpenAI-Compatible API as Reviewer

No OpenAI API? Use DeepSeek, Kimi, MiniMax, GLM, or local models:

```json
{
  "mcpServers": {
    "deepseek-reviewer": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "$DEEPSEEK_API_KEY",
        "LLM_BASE_URL": "https://api.deepseek.com/v1",
        "LLM_MODEL": "deepseek-chat"
      }
    }
  }
}
```

Tested combinations:
- Claude Code (executor) + GPT-5.4 (reviewer) — flagship
- MiniMax-M2.7 (executor) + GLM-5 (reviewer)
- Codex CLI (executor) + Gemini (reviewer) via `gemini-review` MCP
- Any two OpenAI-compatible endpoints

---

## Workflow Overview

```
Workflow 0   /literature-review     → survey existing work
Workflow 1   /idea-discovery        → generate novel research ideas
Workflow 1.5 /experiment-bridge     → run experiments on GPU
Workflow 2   /paper-draft           → write full LaTeX paper
Workflow 3   /auto-review           → multi-round cross-model review + scoring
Workflow 4   /rebuttal              → parse reviews, draft rebuttal, safety-check
             /paper-slides          → Beamer PDF + PPTX + speaker notes
             /paper-poster          → A0/A1 poster PDF + PPTX + SVG
```

---

## Core Commands

### Full Pipeline (Start to Finish)

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

Targeted mode (give it a paper + codebase):

```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

Options:
- `ref paper: <arxiv_url>` — ARIS reads the paper, finds weaknesses, builds on them
- `base repo: <github_url>` — clones repo as base codebase
- `compact: true` — generate lean summary files for short-context models

### Workflow 1 — Idea Discovery

```
/idea-discovery "sparse attention mechanisms for long-context transformers"
```

ARIS will:
1. Survey related work via semantic search
2. Identify open problems and novelty gaps
3. Generate 5–10 research hypotheses
4. Send to reviewer model for adversarial critique
5. Refine surviving ideas into structured proposals

### Workflow 1.5 — Experiment Bridge

```
/experiment-bridge "train sparse attention baseline" — base repo: https://github.com/org/project
```

Parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `code review` | `true` | GPT cross-model code review before GPU deploy |
| `compact` | `false` | Lean output for short-context recovery |
| `wandb` | `true` | W&B experiment tracking via real `wandb.Api()` |

```python
# Example: ARIS auto-generates training scripts like this
import wandb

wandb.init(project="aris-experiment", config={
    "model": "sparse-attn-transformer",
    "dataset": "c4",
    "lr": 3e-4,
    "batch_size": 32,
})

# Training loop ARIS generates and runs autonomously
for step, batch in enumerate(train_loader):
    loss = model(**batch).loss
    loss.backward()
    optimizer.step()
    wandb.log({"train/loss": loss.item(), "step": step})
```

### Workflow 2 — Paper Draft

```
/paper-draft "results/" — venue: NeurIPS, template: neurips2024
```

ARIS auto-selects venue templates:

| Venue | Template |
|-------|----------|
| NeurIPS | `neurips2024.sty` |
| ICML | `icml2024.sty` |
| ICLR | `iclr2024.sty` |
| CVPR | `cvpr2024.sty` |
| ACL | `acl_latex.sty` |
| AAAI | `aaai24.sty` |
| ACM MM | `acmart.cls` |

### Workflow 3 — Auto Review (Cross-Model Loop)

```
/auto-review "paper/" — max rounds: 5, target score: 7
```

Parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max rounds` | `5` | Max review-revision cycles |
| `target score` | `7` | Stop when reviewer scores ≥ this (out of 10) |
| `reviewer model` | MCP default | Override reviewer model per round |

The loop:
1. Claude Code submits paper to reviewer MCP
2. Reviewer returns structured critique + score (1–10)
3. Claude Code revises paper addressing weaknesses
4. Repeat until target score or max rounds

Score progression is logged to `docs/auto_review_score_curve.png`.

### Workflow 4 — Rebuttal

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

Parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | Target venue |
| `character limit` | required | Hard char limit for rebuttal text |
| `quick mode` | `false` | Stop after Phase 0–3 (analysis only, no draft) |
| `auto experiment` | `false` | Run supplementary experiments via `/experiment-bridge` |
| `max stress test rounds` | `1` | GPT-5.4 xhigh stress-test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up round limit |

Three mandatory safety gates — rebuttal will NOT finalize if any fail:
- 🔒 **No fabrication** — every claim maps to paper/review/user-confirmed result
- 🔒 **No overpromise** — every promise is user-approved
- 🔒 **Full coverage** — every reviewer concern tracked

Outputs:
- `PASTE_READY.txt` — exact char count, paste directly to venue portal
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

---

## Slides & Poster Generation

```bash
# Conference slides
/paper-slides "paper/"
# → Beamer PDF + PPTX + speaker notes + Q&A prep

# Conference poster
/paper-poster "paper/"
# → A0/A1 poster PDF (tcbposter) + editable PPTX + SVG
```

---

## Standalone Skills (Use Independently)

Install and invoke individual skills without the full pipeline:

```bash
# Training health check
/training-check "logs/train.log"

# Convert results to paper claims
/result-to-claim "results/table1.csv"

# Plan ablation study
/ablation-planner "method: sparse_attention, baselines: [full_attn, local_attn]"

# Formula derivation and verification
/formula-derivation "derive ELBO for discrete diffusion"

# Grant proposal
/grant-proposal "topic: efficient LLMs for edge devices, agency: NSF"

# Paper illustrations (via Gemini)
/paper-illustration "figure 3: sparse attention pattern"
```

---

## MCP Server: `llm-chat`

The `llm-chat` MCP server is what gives ARIS cross-model review capability. It wraps any OpenAI-compatible chat endpoint as an MCP tool.

```python
# mcp-servers/llm-chat/server.py (simplified interface)
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["LLM_API_KEY"],
    base_url=os.environ.get("LLM_BASE_URL", "https://api.openai.com/v1"),
)

def review_paper(paper_text: str, criteria: dict) -> dict:
    """Called by Claude Code via MCP to get adversarial review."""
    response = client.chat.completions.create(
        model=os.environ.get("LLM_MODEL", "gpt-4o"),
        messages=[
            {"role": "system", "content": "You are a rigorous ML paper reviewer for top-tier venues."},
            {"role": "user", "content": f"Review this paper:\n\n{paper_text}\n\nCriteria: {criteria}"}
        ],
        temperature=0.3,
    )
    return {
        "score": extract_score(response.choices[0].message.content),
        "critique": response.choices[0].message.content,
    }
```

---

## Codex CLI Native Skills

Full skill set also available for OpenAI Codex CLI (no Claude required):

```bash
# Install Codex skills
cp -r skills/skills-codex/ ~/.codex/skills/

# Run pipeline with Codex
codex "/research-pipeline 'attention bottlenecks in vision transformers'"
```

---

## ModelScope Free Tier (Zero Cost)

For zero-cost usage via ModelScope:

```json
{
  "mcpServers": {
    "modelscope-reviewer": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "$MODELSCOPE_API_KEY",
        "LLM_BASE_URL": "https://api-inference.modelscope.cn/v1",
        "LLM_MODEL": "Qwen/Qwen2.5-72B-Instruct"
      }
    }
  }
}
```

See `docs/MODELSCOPE_GUIDE.md` for full setup.

---

## Input Templates

Use templates in `templates/` to structure inputs for each workflow:

```bash
ls templates/
# idea-discovery.md
# experiment-bridge.md
# paper-draft.md
# auto-review.md
# rebuttal.md
# research-pipeline.md
```

Example: `templates/rebuttal.md`

```markdown
## Rebuttal Input

**Paper:** paper/
**Reviews:** reviews/R1.txt, reviews/R2.txt, reviews/R3.txt
**Venue:** NeurIPS
**Character limit:** 5000
**Quick mode:** false
**Auto experiment:** false
```

---

## Compact Mode (Short-Context Recovery)

For short-context models or session recovery after interruption:

```
/research-pipeline "topic" — compact: true
```

Compact mode generates lean `*_summary.md` files instead of full outputs. The `research-refine` workflow auto-saves checkpoints and resumes after interruption.

---

## Troubleshooting

**MCP reviewer not responding**
```bash
# Test MCP server directly
python mcp-servers/llm-chat/server.py --test
# Check env vars are set
echo $LLM_API_KEY
echo $LLM_BASE_URL
```

**Skills not recognized by Claude Code**
```bash
# Verify skill files are in Claude's skill search path
ls ~/.claude/skills/
# Reload Claude Code after adding skills
```

**W&B logging not working**
```bash
pip install wandb
wandb login  # uses $WANDB_API_KEY
# ARIS uses wandb.Api() — ensure project exists
wandb init --project aris-experiment
```

**Cross-model review loop stuck**
- Increase `max rounds` or lower `target score`
- Check reviewer model has sufficient context window for full paper
- Use `compact: true` to reduce token load

**LaTeX compilation errors in paper draft**
```bash
# ARIS auto-runs pdflatex; manually compile to debug:
cd paper/
pdflatex main.tex
bibtex main
pdflatex main.tex && pdflatex main.tex
```

**Anti-hallucination: citation verification fails**
ARIS enforces DBLP → CrossRef → [VERIFY] chain for citations. If a citation can't be verified, it's flagged `[VERIFY]` and will not auto-include. Manually confirm or remove flagged citations before final submission.

---

## Environment Variables Reference

```bash
# OpenAI / Codex reviewer
export OPENAI_API_KEY="..."

# Alternative reviewer APIs
export DEEPSEEK_API_KEY="..."
export MODELSCOPE_API_KEY="..."
export MINIMAX_API_KEY="..."

# Experiment tracking
export WANDB_API_KEY="..."
export WANDB_PROJECT="aris-research"

# Optional: override reviewer model at runtime
export LLM_MODEL="gpt-4o"
export LLM_BASE_URL="https://api.openai.com/v1"
```

---

## Key Directories

```
skills/                    # Core SKILL.md files (one per workflow)
skills/skills-codex/       # Codex CLI native skills
mcp-servers/llm-chat/      # OpenAI-compatible MCP reviewer server
templates/                 # Input templates for each workflow
docs/                      # Adaptation guides (Cursor, Trae, Antigravity, etc.)
docs/CURSOR_ADAPTATION.md
docs/TRAE_ARIS_RUNBOOK_EN.md
docs/ANTIGRAVITY_ADAPTATION.md
docs/MODELSCOPE_GUIDE.md
docs/MiniMax-GLM-Configuration.md
docs/CODEX_GEMINI_REVIEW_GUIDE.md
```

---

## Design Philosophy

- **Two models, not one** — cross-model review breaks self-play blind spots; going 1→2 models is the biggest quality jump
- **Speed × rigor** — fast executor (Claude Code) + deliberate reviewer (GPT/Codex) = better outcomes than either alone
- **Zero lock-in** — every skill is a plain `SKILL.md`; swap any model at any time
- **Methodology, not platform** — the research workflow is the product; take it to any agent
```
