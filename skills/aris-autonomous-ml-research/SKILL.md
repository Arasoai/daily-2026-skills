```markdown
---
name: aris-autonomous-ml-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only autonomous ML research workflows for Claude Code and other LLM agents, enabling cross-model review loops, idea discovery, experiment automation, and paper writing.
triggers:
  - "set up ARIS for autonomous research"
  - "run research pipeline while I sleep"
  - "auto review my paper with ARIS"
  - "use ARIS to generate research ideas"
  - "run experiment bridge with cross-model review"
  - "set up claude code research automation"
  - "write rebuttal automatically with ARIS"
  - "configure ARIS with codex reviewer"
---

# ARIS — Autonomous ML Research in Sleep (Auto-claude-code-research-in-sleep)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous ML research toolkit. Every "skill" is a plain `SKILL.md` file that any LLM agent can read and execute. The system orchestrates **cross-model research loops** — one model executes (Claude Code, Codex, Cursor) while another critiques (GPT-5.4, Gemini, GLM, MiniMax, etc.) — breaking the blind spots of single-model self-review.

No framework. No Docker. No daemon. Just Markdown files and your agent.

---

## Installation

### Option A: Clone the Full Repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

No `pip install` needed — zero Python dependencies for core workflows.

### Option B: Copy Individual Skills

Each skill is self-contained. Copy only what you need:

```bash
# Copy a single skill into your project
cp skills/research-pipeline/SKILL.md .claude/skills/research-pipeline.md
```

### Register Skills with Claude Code

```bash
# In your project root, create the skills directory
mkdir -p .claude/skills

# Symlink or copy the skills you want active
ln -s /path/to/aris/skills/research-pipeline/SKILL.md .claude/skills/research-pipeline.md
ln -s /path/to/aris/skills/experiment-bridge/SKILL.md  .claude/skills/experiment-bridge.md
ln -s /path/to/aris/skills/paper-review/SKILL.md       .claude/skills/paper-review.md
ln -s /path/to/aris/skills/rebuttal/SKILL.md            .claude/skills/rebuttal.md
```

Alternatively, place skills in `~/.claude/skills/` for global availability across all projects.

---

## MCP Server Setup (Cross-Model Review)

ARIS's power comes from routing critique to a *second* model. The bundled `llm-chat` MCP server handles this.

### Install the MCP Server

```bash
cd mcp-servers/llm-chat
pip install -e .
```

### Configure MCP in Claude Code (`~/.claude/mcp.json` or project `.mcp.json`)

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python",
      "args": ["-m", "llm_chat"],
      "cwd": "/path/to/aris/mcp-servers/llm-chat",
      "env": {
        "OPENAI_API_KEY": "${OPENAI_API_KEY}",
        "OPENAI_MODEL": "gpt-4o"
      }
    }
  }
}
```

### Alternative: OpenAI-Compatible Endpoints (No OpenAI Required)

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python",
      "args": ["-m", "llm_chat"],
      "cwd": "/path/to/aris/mcp-servers/llm-chat",
      "env": {
        "OPENAI_API_KEY": "${DEEPSEEK_API_KEY}",
        "OPENAI_BASE_URL": "https://api.deepseek.com/v1",
        "OPENAI_MODEL": "deepseek-chat"
      }
    }
  }
}
```

Supported reviewer models (any OpenAI-compatible API):
- OpenAI: `gpt-4o`, `gpt-4-turbo`
- DeepSeek: set `OPENAI_BASE_URL=https://api.deepseek.com/v1`
- MiniMax: set `OPENAI_BASE_URL=https://api.minimax.chat/v1`
- GLM (ZhipuAI): set `OPENAI_BASE_URL=https://open.bigmodel.cn/api/paas/v4/`
- Kimi: set `OPENAI_BASE_URL=https://api.moonshot.cn/v1`

---

## Core Workflows

### Workflow 0 — Quick Paper Review

Point ARIS at a PDF or directory and get a scored critique:

```
/paper-review "path/to/paper.pdf"
/paper-review "paper/" — venue: NeurIPS, rounds: 3
```

Parameters:
| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | Target venue for scoring rubric |
| `rounds` | `2` | How many review-rewrite cycles |
| `reviewer model` | from MCP config | Override reviewer LLM |

---

### Workflow 1 — Idea Discovery

Generate novel research ideas grounded in literature:

```
/idea-discovery "factorized gap in discrete diffusion LMs"
/idea-discovery "improve contrastive learning for low-resource NLP" — venue: ACL
```

What ARIS does:
1. Pulls recent papers (Semantic Scholar / arXiv)
2. Identifies gaps via cross-model debate
3. Generates 5–10 ranked ideas with novelty + feasibility scores
4. Refines top ideas into problem-anchored proposals

---

### Workflow 1.5 — Experiment Bridge

Run experiments with GPU-safe cross-model code review:

```
/experiment-bridge "train.py" — gpu: A100, wandb: true
/experiment-bridge "experiments/run_ablation.py" — code review: true, compact: true
```

Parameters:
| Parameter | Default | Description |
|-----------|---------|-------------|
| `code review` | `true` | GPT cross-model review before GPU deployment |
| `wandb` | `false` | Enable W&B logging |
| `compact` | `false` | Emit lean summary files for short-context recovery |
| `gpu` | auto-detect | Target GPU spec for resource planning |

---

### Workflow 2 — Full Research Pipeline (End-to-End)

The flagship workflow. Give a direction, wake up to a paper draft:

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

**Targeted mode** — improve an existing paper with its own codebase:

```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

**Options:**
```
/research-pipeline "topic" \
  — venue: NeurIPS \
  — ref paper: https://arxiv.org/abs/XXXX.XXXXX \
  — base repo: https://github.com/org/repo \
  — compact: true \
  — rounds: 3
```

Internal pipeline stages:
1. **Idea Discovery** (`/idea-discovery`)
2. **Literature Grounding** — DBLP → CrossRef → `[VERIFY]` anti-hallucination gate
3. **Experiment Planning** (`/experiment-plan`)
4. **Code Generation + Review** (`/experiment-bridge`)
5. **Result → Claim Mapping** (`/result-to-claim`)
6. **Paper Writing** (LaTeX, venue template)
7. **Cross-Model Review Loop** — score until threshold or max rounds
8. **Ablation Planning** (`/ablation-planner`)

---

### Workflow 3 — Research Refine

Turn vague ideas into submission-ready proposals:

```
/research-refine "my_paper_draft.md"
/research-refine "paper/" — checkpoint: true
```

`checkpoint: true` enables auto-resume after interruption — safe for long overnight runs.

---

### Workflow 4 — Rebuttal (Post-Submission)

Reviews just dropped? ARIS drafts a structured, fabrication-free rebuttal:

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

Full options:
```
/rebuttal "paper_dir/ + review_dir/" \
  — venue: NeurIPS \
  — character limit: 5000 \
  — quick mode: false \
  — auto experiment: true \
  — max stress test rounds: 2 \
  — max followup rounds: 3
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | Venue rules applied to tone + structure |
| `character limit` | **Required** | Hard character cap; enforced on output |
| `quick mode` | `false` | Stop after Phase 0–3 (analysis only, no draft) |
| `auto experiment` | `false` | Auto-run supplementary experiments via `/experiment-bridge` |
| `max stress test rounds` | `1` | GPT stress-test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up cap |

Three safety gates — rebuttal will NOT finalize if any gate fails:
- 🔒 **No fabrication** — every claim maps to paper/review/confirmed result
- 🔒 **No overpromise** — every new commitment is user-approved
- 🔒 **Full coverage** — every reviewer concern is explicitly addressed

Outputs:
- `PASTE_READY.txt` — exact char count, ready to paste to venue system
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

---

### Workflow 5 — Presentation Assets

```
/paper-slides "paper/"     # Beamer PDF + PPTX + speaker notes + Q&A prep
/paper-poster "paper/"     # A0/A1 poster PDF + editable PPTX + SVG
```

Poster options:
```
/paper-poster "paper/" — size: A0, venue: NeurIPS, style: dark
```

---

## Utility Skills

### Standalone Skills (Invoke Directly)

```
/training-check "logs/"           # Detect training pathologies (loss spikes, NaN, etc.)
/result-to-claim "results.json"   # Map experimental results → paper claims
/ablation-planner "paper.md"      # Generate ablation study roadmap
/formula-derivation "method.md"   # Develop and verify mathematical formulations
/paper-illustration "paper/"      # Generate figures (Gemini-powered)
/grant-proposal "idea.md"         # Draft grant proposal from research idea
```

### CitationClaw

Fetch and verify citations without hallucination:

```
/citation-check "paper.tex"       # Verify all \cite{} keys against DBLP + CrossRef
```

---

## Compact Mode (Short-Context / Session Recovery)

For models with limited context windows or long overnight runs:

```
/research-pipeline "topic" — compact: true
```

Compact mode generates `_compact_summary.md` checkpoint files at each stage. To resume after interruption:

```
/research-pipeline "topic" — resume: checkpoints/stage3_compact_summary.md
```

---

## Venue Templates

Pre-configured LaTeX templates for major venues:

```
skills/templates/
├── neurips_2025.sty
├── icml_2025.sty
├── iclr2025_conference.sty
├── cvpr2025.sty
├── acl2025.sty
├── aaai25.sty
└── acmmm2025.sty
```

Reference in your pipeline:

```
/research-pipeline "topic" — venue: CVPR, template: skills/templates/cvpr2025.sty
```

---

## Input Templates

Structured input templates live in `templates/`:

```bash
ls templates/
# research-pipeline-input.md
# experiment-bridge-input.md
# rebuttal-input.md
# paper-review-input.md
```

Copy and fill in the template, then pass it to the skill:

```bash
cp templates/research-pipeline-input.md my_research.md
# edit my_research.md with your topic, constraints, venue
```

```
/research-pipeline my_research.md
```

---

## Alternative Model Combinations (No Claude / No OpenAI)

### MiniMax-M2.7 + GLM-5

```json
{
  "mcpServers": {
    "llm-chat": {
      "env": {
        "OPENAI_API_KEY": "${MINIMAX_API_KEY}",
        "OPENAI_BASE_URL": "https://api.minimax.chat/v1",
        "OPENAI_MODEL": "MiniMax-Text-01"
      }
    }
  }
}
```

Then set Claude Code to use GLM-5 as the executor endpoint. See `docs/MiniMax-GLM-Configuration.md` for full dual-endpoint setup.

### Free Tier via ModelScope

```bash
# No API key needed — uses ModelScope free inference
# See docs/MODELSCOPE_GUIDE.md for full setup
export MODELSCOPE_TOKEN=${MODELSCOPE_TOKEN}
```

---

## Codex CLI Native Skills

Full skill set is available for OpenAI Codex CLI (no Claude required):

```bash
cd skills/skills-codex/
ls
# research-pipeline/SKILL.md
# paper-review/SKILL.md
# experiment-bridge/SKILL.md
# rebuttal/SKILL.md
```

Install into Codex:
```bash
cp skills/skills-codex/research-pipeline/SKILL.md ~/.codex/skills/research-pipeline.md
```

Then invoke in Codex CLI:
```bash
codex "/research-pipeline 'discrete diffusion gaps' -- venue: NeurIPS"
```

---

## Cursor / Trae / Antigravity / Windsurf Adaptation

| IDE | Guide |
|-----|-------|
| Cursor | `docs/CURSOR_ADAPTATION.md` |
| Trae (ByteDance) | `docs/TRAE_ARIS_RUNBOOK_EN.md` |
| Antigravity (Google) | `docs/ANTIGRAVITY_ADAPTATION.md` |
| Codex + Gemini review | `docs/CODEX_GEMINI_REVIEW_GUIDE.md` |
| Alibaba Coding Plan | `docs/ALI_CODING_PLAN_GUIDE.md` |

All adapters follow the same pattern: copy `SKILL.md` files to the IDE's skill/prompt directory, configure the MCP server for the reviewer model, invoke via the IDE's slash-command or prompt interface.

---

## Real Usage Examples

### Example 1: Overnight Research Run

```bash
# 10 PM — kick it off
claude "/research-pipeline 'sparse attention in long-context transformers' \
  -- venue: ICLR \
  -- rounds: 4 \
  -- compact: true"

# 8 AM — check outputs
ls outputs/
# idea_discovery_report.md
# experiment_plan.md
# results/
# paper_draft.tex
# review_rounds/round_1.md ... round_4.md
# paper_draft_final.tex
```

### Example 2: Improve a Specific Paper

```bash
claude "/research-pipeline 'improve discrete diffusion scoring' \
  -- ref paper: https://arxiv.org/abs/2406.04329 \
  -- base repo: https://github.com/kuleshov-group/mdlm \
  -- venue: NeurIPS"
```

### Example 3: Rebuttal Under Deadline

```bash
# Drop reviews into reviews/ directory, paper into paper/
claude "/rebuttal 'paper/ + reviews/' \
  -- venue: ICML \
  -- character limit: 5000 \
  -- quick mode: true"
# → Read strategy report first, then:
claude "/rebuttal 'paper/ + reviews/' \
  -- venue: ICML \
  -- character limit: 5000 \
  -- auto experiment: true"
```

### Example 4: Python Experiment with Cross-Model Code Review

```python
# train.py — your experiment script
import wandb
import torch

def main():
    wandb.init(project="aris-experiment", config={
        "lr": 1e-4,
        "epochs": 50,
        "model": "transformer-base"
    })
    # ... training loop
    wandb.log({"loss": loss.item(), "acc": acc})

if __name__ == "__main__":
    main()
```

```bash
# Let ARIS review the code, then deploy
claude "/experiment-bridge 'train.py' -- code review: true -- wandb: true -- gpu: A100"
```

### Example 5: Custom MCP Bridge (Python)

```python
# mcp-servers/llm-chat/llm_chat/__main__.py pattern
# Extend with your own model endpoint:

import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["OPENAI_API_KEY"],
    base_url=os.environ.get("OPENAI_BASE_URL", "https://api.openai.com/v1")
)

def review(content: str, system_prompt: str) -> str:
    response = client.chat.completions.create(
        model=os.environ.get("OPENAI_MODEL", "gpt-4o"),
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": content}
        ]
    )
    return response.choices[0].message.content
```

---

## Troubleshooting

### MCP Server Not Found
```bash
# Verify the server starts correctly
cd mcp-servers/llm-chat && python -m llm_chat --test
# Check Claude Code sees it
claude mcp list
```

### Skill Not Triggering
```bash
# Confirm skill file is in the right location
ls .claude/skills/        # project-level
ls ~/.claude/skills/      # global
# Skill files must end in .md and contain valid YAML frontmatter
head -20 .claude/skills/research-pipeline.md
```

### Citation Hallucination
ARIS uses a 3-step verification chain: DBLP → CrossRef → `[VERIFY]` flag. If you see `[VERIFY]` tags in output, those citations need manual confirmation. Run:
```
/citation-check "paper.tex"
```

### Compact Mode / Session Recovery
If a long pipeline was interrupted:
```bash
ls outputs/checkpoints/
# stage1_compact_summary.md
# stage3_compact_summary.md  ← resume from latest
claude "/research-pipeline 'topic' -- resume: outputs/checkpoints/stage3_compact_summary.md"
```

### Rebuttal Safety Gate Failure
If the `no fabrication` gate blocks finalization:
```
# ARIS will output a BLOCKED_CLAIMS.md listing unverifiable claims
# Review and either confirm results or remove the claim
# Then re-run with:
claude "/rebuttal 'paper/ + reviews/' -- venue: ICML -- character limit: 5000 -- resume: rebuttal_checkpoint.md"
```

### W&B Integration
```bash
export WANDB_API_KEY=${WANDB_API_KEY}
export WANDB_PROJECT="my-aris-experiments"
# ARIS uses real wandb.Api() calls — ensure your key has write access
wandb login  # one-time setup
```

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENAI_API_KEY` | For OpenAI reviewer | OpenAI or compatible API key |
| `OPENAI_BASE_URL` | Optional | Override endpoint for alternative models |
| `OPENAI_MODEL` | Optional | Reviewer model name (default: `gpt-4o`) |
| `ANTHROPIC_API_KEY` | For Claude executor | Set in Claude Code config |
| `WANDB_API_KEY` | For W&B logging | Weights & Biases API key |
| `MODELSCOPE_TOKEN` | For free tier | ModelScope inference token |

---

## Project Structure

```
Auto-claude-code-research-in-sleep/
├── skills/
│   ├── research-pipeline/SKILL.md   # Workflow 2: end-to-end
│   ├── paper-review/SKILL.md        # Workflow 0: review + score
│   ├── idea-discovery/SKILL.md      # Workflow 1: idea generation
│   ├── experiment-bridge/SKILL.md   # Workflow 1.5: run experiments
│   ├── research-refine/SKILL.md     # Workflow 3: refine proposals
│   ├── rebuttal/SKILL.md            # Workflow 4: post-submission
│   ├── paper-slides/SKILL.md        # Workflow 5a: slides
│   ├── paper-poster/SKILL.md        # Workflow 5b: poster
│   ├── training-check/SKILL.md      # Utility: detect training issues
│   ├── result-to-claim/SKILL.md     # Utility: results → claims
│   ├── ablation-planner/SKILL.md    # Utility: ablation roadmap
│   ├── formula-derivation/SKILL.md  # Utility: math derivation
│   ├── paper-illustration/SKILL.md  # Utility: figure generation
│   ├── grant-proposal/SKILL.md      # Utility: grant writing
│   └── skills-codex/                # Codex CLI native versions
├── mcp-servers/
│   └── llm-chat/                    # OpenAI-compatible review bridge
├── templates/                       # Input templates for each workflow
├── docs/
│   ├── CURSOR_ADAPTATION.md
│   ├── TRAE_ARIS_RUNBOOK_EN.md
│   ├── ANTIGRAVITY_ADAPTATION.md
│   ├── CODEX_GEMINI_REVIEW_GUIDE.md
│   ├── MODELSCOPE_GUIDE.md
│   └── MiniMax-GLM-Configuration.md
└── README.md
```
```
