```markdown
---
name: aris-autonomous-ml-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only skill system for autonomous ML research workflows using cross-model collaboration, paper review loops, idea discovery, and experiment automation with Claude Code, Codex, or any LLM agent.
triggers:
  - "set up ARIS for autonomous research"
  - "run research pipeline while I sleep"
  - "automate ML paper writing with Claude Code"
  - "cross-model paper review loop"
  - "use ARIS to find research ideas"
  - "run experiment bridge for GPU experiments"
  - "write rebuttal for paper reviews"
  - "install ARIS skills for Claude Code"
---

# ARIS — Autonomous ML Research In Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous research system. Every "skill" is a plain `SKILL.md` file that any LLM agent can read and execute. The core insight: Claude Code executes fast, an external model (GPT, Gemini, GLM, MiniMax, etc.) acts as a rigorous cross-model reviewer — breaking the self-review blind spot that kills single-model research loops.

**What it can do autonomously:**
- Discover novel research ideas from a direction or existing paper
- Search and verify related work (DBLP → CrossRef → fallback)
- Generate experiment plans and run them via GPU bridges
- Write full LaTeX papers with venue templates
- Review and iteratively improve paper scores (tracked in `auto_review_score_curve.png`)
- Draft conference rebuttals with safety gates (no fabrication, no overpromise)
- Generate slides (Beamer/PPTX) and posters (A0/A1)

---

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (primary agent) OR Codex CLI / Cursor / Trae / Antigravity
- An OpenAI-compatible API for the reviewer model (OpenAI, MiniMax, GLM, Kimi, DeepSeek, etc.)

### Clone and install skills

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep

# Install all Claude Code skills at once
cp -r skills/. ~/.claude/skills/
# Or install individually:
cp skills/research-pipeline/SKILL.md ~/.claude/skills/research-pipeline.md
cp skills/experiment-bridge/SKILL.md ~/.claude/skills/experiment-bridge.md
cp skills/auto-review/SKILL.md ~/.claude/skills/auto-review.md
cp skills/rebuttal/SKILL.md ~/.claude/skills/rebuttal.md
```

### Set up the reviewer MCP server (Codex MCP or llm-chat)

```bash
# Option A: OpenAI Codex as reviewer
export OPENAI_API_KEY="$OPENAI_API_KEY"
# Configure in claude_desktop_config.json or .mcp.json

# Option B: Any OpenAI-compatible API (GLM, MiniMax, Kimi, DeepSeek, etc.)
cd mcp-servers/llm-chat
pip install -r requirements.txt
# Set your provider's base URL and key:
export LLM_CHAT_BASE_URL="https://api.minimax.chat/v1"
export LLM_CHAT_API_KEY="$MINIMAX_API_KEY"
export LLM_CHAT_MODEL="MiniMax-M2.7"
python server.py
```

### MCP configuration (`.mcp.json` in project root)

```json
{
  "mcpServers": {
    "codex": {
      "command": "npx",
      "args": ["-y", "@openai/codex-mcp"],
      "env": {
        "OPENAI_API_KEY": "${OPENAI_API_KEY}"
      }
    },
    "llm-chat": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_CHAT_BASE_URL": "${LLM_CHAT_BASE_URL}",
        "LLM_CHAT_API_KEY": "${LLM_CHAT_API_KEY}",
        "LLM_CHAT_MODEL": "${LLM_CHAT_MODEL}"
      }
    }
  }
}
```

---

## Core Workflows

### Workflow 0 — Full pipeline from direction to paper

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

Runs all sub-workflows sequentially: idea discovery → literature review → experiment plan → experiment bridge → paper writing → auto-review loop.

**With a reference paper and base repo (targeted improvement mode):**

```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

ARIS reads the paper → finds its weaknesses → clones the codebase → generates ideas that fix those weaknesses using that code.

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ref paper` | — | arXiv URL to improve upon |
| `base repo` | — | GitHub repo to use as codebase |
| `venue` | `NeurIPS` | Target venue for paper formatting |
| `compact` | `false` | Generate lean summaries for short-context models |

---

### Workflow 1 — Idea discovery only

```
/idea-discovery "sparse mixture-of-experts for long-context reasoning"
```

Produces `IDEAS.md` with ranked ideas, novelty scores, and feasibility assessments. Each idea links to weaknesses found in related work.

---

### Workflow 1.5 — Experiment bridge (GPU execution)

```
/experiment-bridge "train sparse MoE baseline" — code review: true
```

Before deploying to GPU, GPT-5.4 (or your reviewer model) cross-reviews the generated code. Set `code review: false` to skip.

```python
# experiment-bridge reads your experiment plan and generates:
# 1. A self-contained train.py with W&B logging
# 2. A GPU job script (SLURM or local)
# 3. A results parser that feeds back into the paper

# Example generated train.py structure:
import wandb
import torch

def main(config):
    wandb.init(project="aris-exp", config=config)
    
    model = build_model(config)
    for epoch in range(config["epochs"]):
        loss = train_epoch(model, config)
        wandb.log({"loss": loss, "epoch": epoch})
    
    # ARIS reads wandb.Api() results back into paper claims
    api = wandb.Api()
    run = api.run(f"aris-exp/{wandb.run.id}")
    return run.summary
```

---

### Workflow 2 — Literature review with anti-hallucination

```
/literature-review "discrete diffusion language models"
```

Enforces: DBLP lookup → CrossRef verification → `[VERIFY]` tag for unconfirmed citations. Never fabricates paper titles or authors.

---

### Workflow 3 — Paper writing

```
/paper-write "paper/" — venue: NeurIPS
```

Supported venue templates: `NeurIPS`, `ICML`, `ICLR`, `CVPR`, `ACL`, `AAAI`, `ACM MM`. Generates full LaTeX in `paper/` with proper venue formatting.

---

### Workflow 4 — Rebuttal (post-submission)

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

Three mandatory safety gates — rebuttal will NOT finalize if any fails:
- 🔒 **No fabrication** — every claim maps to paper/review/confirmed result
- 🔒 **No overpromise** — every promise is user-approved
- 🔒 **Full coverage** — every reviewer concern addressed

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | Target venue (affects formatting/limits) |
| `character limit` | required | Hard limit for final rebuttal text |
| `quick mode` | `false` | Stop after parsing + strategy (no draft) |
| `auto experiment` | `false` | Auto-run supplementary experiments |
| `max stress test rounds` | `1` | GPT-5.4 adversarial stress-test passes |
| `max followup rounds` | `3` | Per-reviewer follow-up limit |

**Outputs:**
- `PASTE_READY.txt` — exact character count, ready to paste to venue
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

---

### Workflow 5 — Auto-review loop

```
/auto-review "paper/"
```

Runs iterative review rounds. Tracks scores in `auto_review_score_curve.png`. Stops when score plateaus or hits target.

---

### Presentation outputs

```bash
# Conference slides
/paper-slides "paper/"
# → Beamer PDF + PPTX + speaker notes + Q&A prep

# Conference poster
/paper-poster "paper/"
# → A0/A1 poster PDF + editable PPTX + SVG
# → Uses tcbposter, venue colors, visual review
```

---

## Alternative Model Combinations

ARIS works with any OpenAI-compatible API. No Claude or OpenAI required.

```bash
# MiniMax-M2.7 + GLM-5
export LLM_CHAT_BASE_URL="https://api.minimax.chat/v1"
export LLM_CHAT_API_KEY="$MINIMAX_API_KEY"
export LLM_CHAT_MODEL="MiniMax-M2.7"

# Kimi K2.5 as reviewer
export LLM_CHAT_BASE_URL="https://api.moonshot.cn/v1"
export LLM_CHAT_API_KEY="$KIMI_API_KEY"
export LLM_CHAT_MODEL="moonshot-v1-128k"

# DeepSeek as reviewer
export LLM_CHAT_BASE_URL="https://api.deepseek.com/v1"
export LLM_CHAT_API_KEY="$DEEPSEEK_API_KEY"
export LLM_CHAT_MODEL="deepseek-chat"

# Free tier via ModelScope (zero cost)
# See docs/MODELSCOPE_GUIDE.md
```

---

## Alternative Agent Environments

| Agent | Guide |
|-------|-------|
| Codex CLI | `skills/skills-codex/` — full skill set |
| Cursor | `docs/CURSOR_ADAPTATION.md` |
| Trae (ByteDance) | `docs/TRAE_ARIS_RUNBOOK_EN.md` |
| Antigravity (Google) | `docs/ANTIGRAVITY_ADAPTATION.md` |
| Windsurf | Works with standard SKILL.md install |

---

## Community Utility Skills

Standalone skills that work independently or integrate into core workflows:

```bash
# Formula derivation and verification
/formula-derivation "derive ELBO for discrete diffusion"

# Training diagnostics (detects loss spikes, dead neurons, etc.)
/training-check "runs/train_log.txt"

# Convert raw results to paper claims
/result-to-claim "results/exp_001.json"

# Plan ablation study
/ablation-planner "model components: sparse-attn, rope, gating"

# Gemini-powered paper illustration
/paper-illustration "paper/figures/"

# Citation graph via CitationClaw
/citation-claw "paper.bib"

# Grant proposal writing
/grant-proposal "research direction: ..."
```

---

## Session Recovery and Compact Mode

For long sessions or short-context models, use `compact` mode to generate lean summary files that allow resuming:

```
/research-pipeline "topic" — compact: true
```

For `research-refine` checkpoint recovery (auto-resumes after interruption):

```bash
# ARIS writes checkpoint files to:
# .aris/checkpoints/research-refine-{timestamp}.json
# On next run with same topic, it auto-detects and resumes
/research-refine "topic"  # detects existing checkpoint automatically
```

---

## Project Structure

```
Auto-claude-code-research-in-sleep/
├── skills/                      # Core SKILL.md files
│   ├── research-pipeline/       # Workflow 0: full pipeline
│   ├── idea-discovery/          # Workflow 1
│   ├── experiment-bridge/       # Workflow 1.5
│   ├── literature-review/       # Workflow 2
│   ├── paper-write/             # Workflow 3
│   ├── rebuttal/                # Workflow 4
│   ├── auto-review/             # Workflow 5
│   ├── paper-slides/
│   ├── paper-poster/
│   ├── training-check/
│   ├── result-to-claim/
│   ├── ablation-planner/
│   ├── formula-derivation/
│   ├── research-refine/
│   ├── experiment-plan/
│   └── skills-codex/            # Codex CLI native versions
├── mcp-servers/
│   └── llm-chat/                # Any OpenAI-compatible reviewer
├── templates/                   # Input templates for every workflow
├── docs/                        # Adaptation guides per IDE
└── .aris/checkpoints/           # Auto-generated session state
```

---

## Troubleshooting

**Reviewer model not responding:**
```bash
# Test llm-chat MCP server directly
python mcp-servers/llm-chat/server.py --test
# Check env vars are set
echo $LLM_CHAT_BASE_URL $LLM_CHAT_MODEL
```

**W&B logging broken in experiment-bridge:**
```python
# ARIS requires real wandb.Api() calls — ensure you're logged in:
import wandb
wandb.login()  # uses $WANDB_API_KEY env var
# Do NOT mock wandb in experiment scripts — ARIS reads real run data
```

**Literature review hallucinating citations:**
```
# Add explicit instruction to skill invocation:
/literature-review "topic" — strict citations: true
# This enforces DBLP → CrossRef → [VERIFY] chain
# Any unverified citation gets [VERIFY] tag, never fabricated
```

**Rebuttal exceeds character limit:**
```
# Use quick mode first to see reviewer concerns without drafting:
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000, quick mode: true
# Then selectively address high-priority concerns in full mode
```

**Session interrupted mid-pipeline:**
```bash
# Check for checkpoint files:
ls .aris/checkpoints/
# Re-run the same command — ARIS detects and resumes from checkpoint
/research-pipeline "same topic as before"
```

**Short context model running out of tokens:**
```
# Use compact mode to generate lean summaries at each stage:
/research-pipeline "topic" — compact: true
# Generates *_compact.md files alongside full outputs
```

---

## Key Design Principles

1. **Zero dependencies** — every skill is a plain `.md` file; no pip install, no Docker
2. **Cross-model review** — executor and reviewer are always different models to avoid blind spots
3. **Anti-hallucination by default** — literature review enforces multi-source verification
4. **Safety gates in rebuttal** — fabrication/overpromise/coverage checks are non-negotiable
5. **Swap anything** — change agent IDE, reviewer model, or GPU backend without rewriting workflows
6. **Methodology over platform** — ARIS is a research workflow pattern; take it to any stack
```
