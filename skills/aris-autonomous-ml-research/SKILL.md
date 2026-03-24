```markdown
---
name: aris-autonomous-ml-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only skill workflows for autonomous ML research: cross-model review loops, idea discovery, experiment automation, paper writing, and rebuttal generation using Claude Code, Codex, or any LLM agent.
triggers:
  - "set up ARIS for autonomous research"
  - "run research pipeline while I sleep"
  - "automate ML paper writing with Claude Code"
  - "cross-model review loop for my paper"
  - "generate research ideas automatically"
  - "run experiment bridge with ARIS"
  - "write rebuttal for my paper reviews"
  - "install ARIS skills in Claude Code"
---

# ARIS — Autonomous ML Research In Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous ML research system. It orchestrates cross-model workflows where one LLM executes research and another acts as adversarial reviewer — breaking the self-play blind spot. The entire system is plain `.md` files: no framework, no database, no daemon, no lock-in.

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### 2. Install Skills into Claude Code

Copy skill files into your Claude Code custom commands directory:

```bash
# For Claude Code (default location)
mkdir -p ~/.claude/commands
cp -r skills/research-pipeline/SKILL.md ~/.claude/commands/research-pipeline.md
cp -r skills/experiment-bridge/SKILL.md ~/.claude/commands/experiment-bridge.md
cp -r skills/paper-polish/SKILL.md ~/.claude/commands/paper-polish.md
cp -r skills/auto-review/SKILL.md ~/.claude/commands/auto-review.md
cp -r skills/rebuttal/SKILL.md ~/.claude/commands/rebuttal.md
cp -r skills/idea-discovery/SKILL.md ~/.claude/commands/idea-discovery.md
cp -r skills/paper-slides/SKILL.md ~/.claude/commands/paper-slides.md
cp -r skills/paper-poster/SKILL.md ~/.claude/commands/paper-poster.md

# Or install ALL skills at once
for dir in skills/*/; do
  name=$(basename "$dir")
  cp "$dir/SKILL.md" ~/.claude/commands/"$name".md
done
```

### 3. Configure the Cross-Model Reviewer (Codex MCP)

ARIS uses a second model as adversarial reviewer. The default setup is **Claude Code (executor) + Codex/GPT-5.4 (reviewer)** via MCP.

```bash
# Install Codex MCP server
npm install -g @openai/codex-mcp

# Add to your Claude Code MCP config (~/.claude/mcp_config.json)
```

```json
{
  "mcpServers": {
    "codex": {
      "command": "codex-mcp",
      "args": ["--model", "gpt-5.4-xhigh"],
      "env": {
        "OPENAI_API_KEY": "$OPENAI_API_KEY"
      }
    }
  }
}
```

### 4. (Optional) Custom LLM Reviewer via llm-chat MCP

Use any OpenAI-compatible API as reviewer — no Claude or OpenAI required:

```bash
cd mcp-servers/llm-chat
pip install -r requirements.txt
```

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "$LLM_API_KEY",
        "LLM_BASE_URL": "https://api.deepseek.com/v1",
        "LLM_MODEL": "deepseek-chat"
      }
    }
  }
}
```

Tested reviewer models: `GLM-5`, `MiniMax-M2.7`, `Kimi-K2.5`, `LongCat`, `DeepSeek`, `Gemini`.

### 5. Free Tier via ModelScope

```bash
# Zero cost, zero lock-in
export MODELSCOPE_API_KEY=$MODELSCOPE_API_KEY
# See docs/MODELSCOPE_GUIDE.md for full setup
```

---

## Core Concepts

| Concept | What it means |
|---|---|
| **Skill** | A single `SKILL.md` file that tells any LLM agent how to perform a research workflow |
| **Executor** | The primary agent (Claude Code, Codex CLI, Cursor, Trae, etc.) that does the work |
| **Reviewer** | A *different* LLM called via MCP that adversarially critiques the executor's output |
| **Cross-model loop** | Executor → output → Reviewer critique → Executor revision → repeat until score threshold |
| **Zero lock-in** | Swap any model at any time — the Markdown skills work regardless of which agent reads them |

---

## Workflows

### Workflow 0 — Full Autonomous Pipeline

Give ARIS a research direction. It handles ideation → experiments → writing → review → polish.

```
# In Claude Code chat:
/research-pipeline "factorized gap in discrete diffusion LMs"

# With a reference paper and base codebase:
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

**Parameters:**

| Parameter | Default | Description |
|---|---|---|
| `ref paper` | — | arXiv URL to read, find weaknesses, and fix |
| `base repo` | — | GitHub repo to clone as starting codebase |
| `venue` | `ICML` | Target conference (ICML/NeurIPS/ICLR/CVPR/ACL/AAAI) |
| `max review rounds` | `3` | How many cross-model review iterations to run |
| `compact` | `false` | Generate lean summary files for short-context models |

---

### Workflow 1 — Idea Discovery

```
/idea-discovery "sparse mixture of experts efficiency"
```

Generates a scored list of research ideas with novelty, feasibility, and impact ratings. Each idea includes: problem statement, proposed method, baseline comparisons, expected results.

---

### Workflow 1.5 — Experiment Bridge

Run experiments with cross-model code review before GPU deployment:

```
/experiment-bridge "train.py with config experiments/ablation_v2.yaml"

# Disable pre-deployment code review (not recommended):
/experiment-bridge "train.py" — code review: false

# With W&B tracking:
/experiment-bridge "train.py" — wandb project: my-research, wandb entity: myteam
```

The bridge:
1. Claude Code reads your experiment config
2. GPT-5.4 xhigh reviews the code for bugs, numerical instability, and logic errors
3. You approve/reject the code review findings
4. Experiment runs; results are parsed into structured claims

```python
# Example: ARIS-compatible experiment script structure
# experiments/train.py

import wandb
import argparse

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--config", type=str, required=True)
    parser.add_argument("--exp_name", type=str, default="aris_exp")
    args = parser.parse_args()

    # ARIS reads this run config
    wandb.init(
        project=os.environ.get("WANDB_PROJECT", "aris-research"),
        name=args.exp_name,
        config=load_yaml(args.config)
    )

    # ... training loop ...

    # ARIS parses these metrics automatically
    wandb.log({
        "val_loss": val_loss,
        "val_acc": val_acc,
        "epoch": epoch
    })

    # ARIS uses this as final claim evidence
    wandb.summary["best_val_acc"] = best_acc
    wandb.finish()
```

---

### Workflow 2 — Paper Writing

```
/paper-polish "paper/" — venue: NeurIPS
```

Reads your paper directory, runs cross-model review, and iteratively improves:
- Clarity and narrative flow
- Related work with verified citations (DBLP → CrossRef → `[VERIFY]` fallback — never hallucinated)
- Abstract, intro, conclusion alignment
- Section-by-section polish with reviewer feedback

---

### Workflow 3 — Auto Review Loop

```
/auto-review "paper/draft_v3.pdf" — target score: 7, max rounds: 5
```

Produces the score progression curve (like `docs/auto_review_score_curve.png`). Each round:
1. Claude Code reads current draft
2. Codex reviews and scores (1–10) with specific weaknesses
3. Claude Code revises based on critique
4. Repeat until target score or max rounds

---

### Workflow 4 — Rebuttal Generation

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

**Parameters:**

| Parameter | Default | Description |
|---|---|---|
| `venue` | `ICML` | ICML/NeurIPS/ICLR/CVPR/ACL/AAAI/ACM |
| `character limit` | required | Hard char limit for rebuttal text |
| `quick mode` | `false` | Stop after Phase 0–3 (parse + strategy only) |
| `auto experiment` | `false` | Auto-run supplementary experiments when reviewers request new evidence |
| `max stress test rounds` | `1` | GPT-5.4 stress-test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up round limit |

**Three safety gates** — rebuttal will NOT finalize if any fails:
- 🔒 **No fabrication** — every claim maps to paper/review/user-confirmed result
- 🔒 **No overpromise** — every promise is user-approved  
- 🔒 **Full coverage** — every reviewer concern is tracked

**Outputs:**
- `PASTE_READY.txt` — exact char count, paste directly to venue system
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

```
# Quick mode — understand what reviewers want before writing
/rebuttal "paper/ + reviews" — venue: NeurIPS, character limit: 5000, quick mode: true

# With auto supplementary experiments
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000, auto experiment: true
```

---

### Workflow 5 — Presentation Generation

```
# Conference slides (Beamer PDF + PPTX + speaker notes + Q&A prep)
/paper-slides "paper/"

# Conference poster (A0/A1 PDF + PPTX + SVG, venue colors)
/paper-poster "paper/" — venue: NeurIPS, size: A0
```

---

## Standalone Utility Skills

```
# Formula derivation and verification
/formula-derivation "derive the ELBO for hierarchical VAE with discrete latents"

# Grant proposal writing
/grant-proposal "paper/" — funder: NSF, program: IIS

# Paper illustration with Gemini
/paper-illustration "figure 3 showing attention flow in transformer"

# Ablation planning
/ablation-planner "experiments/results.json"

# Training health check
/training-check "logs/train_log.txt"

# Result-to-claim conversion
/result-to-claim "experiments/results.json"
```

---

## Alternative Model Combinations

No Claude or OpenAI API required. Configure any pair:

```bash
# MiniMax-M2.7 executor + GLM-5 reviewer
export EXECUTOR_API_KEY=$MINIMAX_API_KEY
export EXECUTOR_BASE_URL="https://api.minimax.chat/v1"
export EXECUTOR_MODEL="MiniMax-M2.7"

export REVIEWER_API_KEY=$GLM_API_KEY
export REVIEWER_BASE_URL="https://open.bigmodel.cn/api/paas/v4"
export REVIEWER_MODEL="glm-5"

# See docs/MiniMax-GLM-Configuration.md for full setup
```

```bash
# Kimi-K2.5 + DeepSeek
export EXECUTOR_MODEL="kimi-k2.5"
export REVIEWER_MODEL="deepseek-chat"

# Codex executor + Gemini reviewer
# See docs/CODEX_GEMINI_REVIEW_GUIDE.md
```

---

## Alternative Agent Adapters

| Agent | Guide |
|---|---|
| **Codex CLI** | `skills/skills-codex/` — full skill set |
| **Cursor** | `docs/CURSOR_ADAPTATION.md` |
| **Trae** (ByteDance) | `docs/TRAE_ARIS_RUNBOOK_EN.md` |
| **Antigravity** (Google) | `docs/ANTIGRAVITY_ADAPTATION.md` |
| **Windsurf** | Native SKILL.md support |

---

## Project Structure

```
Auto-claude-code-research-in-sleep/
├── skills/
│   ├── research-pipeline/SKILL.md   # Full pipeline
│   ├── idea-discovery/SKILL.md      # Workflow 1
│   ├── experiment-bridge/SKILL.md   # Workflow 1.5
│   ├── paper-polish/SKILL.md        # Workflow 2
│   ├── auto-review/SKILL.md         # Workflow 3
│   ├── rebuttal/SKILL.md            # Workflow 4
│   ├── paper-slides/SKILL.md        # Workflow 5a
│   ├── paper-poster/SKILL.md        # Workflow 5b
│   ├── formula-derivation/SKILL.md
│   ├── grant-proposal/SKILL.md
│   ├── ablation-planner/SKILL.md
│   ├── training-check/SKILL.md
│   ├── result-to-claim/SKILL.md
│   ├── research-refine/SKILL.md
│   ├── experiment-plan/SKILL.md
│   └── skills-codex/                # Codex CLI native versions
├── mcp-servers/
│   └── llm-chat/                    # Any OpenAI-compatible reviewer
│       └── server.py
├── templates/                       # Input templates for every workflow
│   ├── research-pipeline.md
│   ├── rebuttal.md
│   └── ...
├── docs/
│   ├── CURSOR_ADAPTATION.md
│   ├── TRAE_ARIS_RUNBOOK_EN.md
│   ├── ANTIGRAVITY_ADAPTATION.md
│   ├── CODEX_GEMINI_REVIEW_GUIDE.md
│   ├── MiniMax-GLM-Configuration.md
│   ├── MODELSCOPE_GUIDE.md
│   └── ALI_CODING_PLAN_GUIDE.md
└── README.md
```

---

## Using Input Templates

Templates prevent missed parameters:

```bash
# Copy the template for your workflow
cp templates/research-pipeline.md my_research_run.md

# Edit with your specifics
cat my_research_run.md
```

```markdown
# Research Pipeline Input

direction: "factorized gap in discrete diffusion LMs"
ref paper: https://arxiv.org/abs/2406.04329
base repo: https://github.com/org/discrete-diffusion
venue: NeurIPS
max review rounds: 4
compact: false
```

```
# Then invoke with the filled template
/research-pipeline @my_research_run.md
```

---

## Compact Mode (Short-Context Models)

When using models with limited context windows or resuming interrupted sessions:

```
/research-pipeline "my topic" — compact: true
```

Compact mode generates lean `*_summary.md` files at each phase checkpoint instead of full verbose outputs. Use for:
- Models with < 32k context
- Resuming after session interruption (checkpoint auto-resume)
- Reducing API cost on long pipelines

---

## Troubleshooting

### Skill not found in Claude Code

```bash
# Verify installation path
ls ~/.claude/commands/
# Should list your .md files

# Check Claude Code version supports custom commands
claude --version
```

### MCP reviewer not connecting

```bash
# Test Codex MCP directly
codex-mcp --test

# Check MCP config syntax
cat ~/.claude/mcp_config.json | python -m json.tool

# Verify API key is set
echo $OPENAI_API_KEY | cut -c1-8  # Should show first 8 chars
```

### Cross-model review loop not triggering

The reviewer is only called when Claude Code has MCP access. Ensure:
1. MCP server is listed in `~/.claude/mcp_config.json`
2. The MCP server process can start (run `codex-mcp` standalone to test)
3. You're running Claude Code in a project directory (not a bare terminal)

### Citation hallucination in paper writing

ARIS Workflow 2 enforces `DBLP → CrossRef → [VERIFY]` — if a citation cannot be confirmed it is marked `[VERIFY]` rather than hallucinated. If you see raw `[VERIFY]` tags in output:

```bash
# Manually verify and fill in:
grep -n "\[VERIFY\]" paper/related_work.tex
# Then look up each on https://dblp.org or https://crossref.org
```

### Rebuttal exceeds character limit

```
# Check exact count before finalizing
wc -c PASTE_READY.txt

# If over limit, add tighter constraint:
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 4800
# Use 4800 instead of 5000 as safety margin
```

### W&B integration in experiment-bridge

```bash
# Ensure wandb is authenticated
wandb login  # Uses $WANDB_API_KEY or interactive login

# ARIS uses real wandb.Api() calls — verify API access
python -c "import wandb; api = wandb.Api(); print('W&B OK')"
```

### Session interrupted mid-pipeline

```
# research-refine has auto-checkpoint — just re-run the same command
# ARIS detects the checkpoint and resumes from last completed phase
/research-pipeline "my topic"  # Will resume automatically
```

---

## Environment Variables Reference

```bash
# OpenAI / Codex reviewer
export OPENAI_API_KEY="..."

# Custom LLM reviewer (llm-chat MCP)
export LLM_API_KEY="..."
export LLM_BASE_URL="https://api.deepseek.com/v1"
export LLM_MODEL="deepseek-chat"

# ModelScope (free tier)
export MODELSCOPE_API_KEY="..."

# Weights & Biases
export WANDB_API_KEY="..."
export WANDB_PROJECT="aris-research"
export WANDB_ENTITY="myteam"

# MiniMax
export MINIMAX_API_KEY="..."

# Zhipu GLM
export GLM_API_KEY="..."

# Kimi
export KIMI_API_KEY="..."
```

---

## Quick Reference Card

```
# Full pipeline (everything)
/research-pipeline "topic"

# Just ideas
/idea-discovery "topic"

# Run experiments with code review
/experiment-bridge "script.py"

# Polish existing paper
/paper-polish "paper/" — venue: ICML

# Automated review loop
/auto-review "paper/draft.pdf" — target score: 7

# Rebuttal from reviews
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000

# Slides + poster
/paper-slides "paper/"
/paper-poster "paper/"
```
```
