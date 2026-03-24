```markdown
---
name: aris-autonomous-ml-research
description: Autonomous ML research workflows using ARIS (Auto-Research-In-Sleep) — Markdown-only skills for cross-model paper review, idea discovery, experiment automation, and paper writing with Claude Code, Codex, or any LLM agent.
triggers:
  - "set up ARIS for autonomous research"
  - "run research pipeline while I sleep"
  - "automate ML paper writing with Claude Code"
  - "cross-model review loop for my paper"
  - "use ARIS to find research ideas"
  - "run experiment automation with ARIS"
  - "set up auto paper review workflow"
  - "write rebuttal with ARIS"
---

# ARIS — Auto-Research-In-Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous ML research system. Every "skill" is a plain `SKILL.md` file that any LLM agent can read and execute. It orchestrates cross-model collaboration — one model executes research (Claude Code, Codex, etc.) while another acts as adversarial reviewer (GPT-5.4, Gemini, GLM, MiniMax, etc.) to break self-play blind spots.

**Core value**: going from research direction → paper ideas → experiments → written paper → rebuttal, autonomously, overnight.

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

No `pip install`, no Docker, no daemon. The entire system is Markdown files.

### 2. Install Claude Code (Primary Agent)

```bash
npm install -g @anthropic-ai/claude-code
```

### 3. Install Codex MCP (Cross-Model Reviewer)

```bash
npm install -g @openai/codex
```

Configure Claude Code to use the Codex MCP server by adding to your Claude Code config (`~/.claude/settings.json`):

```json
{
  "mcpServers": {
    "codex": {
      "command": "codex",
      "args": ["mcp"],
      "env": {
        "OPENAI_API_KEY": "$OPENAI_API_KEY"
      }
    }
  }
}
```

### 4. Copy Skills into Claude Code

```bash
# Copy all skills to Claude Code's custom skills directory
cp -r skills/claude-code/ ~/.claude/skills/

# Or symlink to stay up to date
ln -s $(pwd)/skills/claude-code ~/.claude/skills/aris
```

### 5. Set Environment Variables

```bash
# Required for Claude Code
export ANTHROPIC_API_KEY=your_anthropic_key

# Required for cross-model review (GPT-5.4 as reviewer)
export OPENAI_API_KEY=your_openai_key

# Optional: alternative reviewer models (no OpenAI needed)
export LLM_REVIEWER_BASE_URL=https://api.minimax.chat/v1
export LLM_REVIEWER_API_KEY=your_minimax_key
export LLM_REVIEWER_MODEL=MiniMax-M2.7
```

---

## Alternative Model Combinations (No Claude/OpenAI Required)

ARIS works with any OpenAI-compatible API. Configure the `llm-chat` MCP server:

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "node",
      "args": ["mcp-servers/llm-chat/index.js"],
      "env": {
        "LLM_BASE_URL": "$LLM_REVIEWER_BASE_URL",
        "LLM_API_KEY": "$LLM_REVIEWER_API_KEY",
        "LLM_MODEL": "$LLM_REVIEWER_MODEL"
      }
    }
  }
}
```

Tested combinations:
| Executor | Reviewer | Config |
|----------|----------|--------|
| Claude Code | GPT-5.4 xhigh | Default |
| Codex CLI | Gemini | [Guide](docs/CODEX_GEMINI_REVIEW_GUIDE.md) |
| Claude Code | MiniMax-M2.7 | `LLM_BASE_URL=https://api.minimax.chat/v1` |
| Claude Code | GLM-5 | `LLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4` |
| MiniMax-M2.7 | GLM-5 | [Guide](docs/MiniMax-GLM-Configuration.md) |
| Codex CLI | Claude | Swap executor/reviewer |

---

## Core Workflows

### Workflow 0: Full Pipeline (Start Here)

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

**With a reference paper and base repo:**
```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

ARIS will:
1. Read the paper → find weaknesses
2. Clone the codebase
3. Generate ideas that fix those weaknesses using that code
4. Run experiments
5. Write the paper

**Parameters:**
```
/research-pipeline "topic"
  — ref paper: <arxiv_url>       # Optional: paper to improve
  — base repo: <github_url>      # Optional: codebase to build on
  — venue: ICML                  # Target venue (default: ICML)
  — compact: true                # Lean summaries for short-context models
```

---

### Workflow 1: Idea Discovery

```
/idea-discovery "discrete diffusion language models"
```

Scans literature, identifies gaps, generates novel research directions, scores each idea for novelty/feasibility, and outputs a ranked proposal list.

---

### Workflow 1.5: Experiment Bridge

```
/experiment-bridge "run ablation on temperature scaling" — code review: true
```

Cross-model code review before GPU deployment (enabled by default). Catches bugs, confirms experimental validity, then runs.

```python
# Example: what experiment-bridge automates
# 1. Claude Code writes training script
# 2. GPT-5.4 reviews the code (code review gate)
# 3. If approved → submits to GPU cluster
# 4. Monitors via W&B API
import wandb

api = wandb.Api()
runs = api.runs("your-entity/your-project")
for run in runs:
    print(run.name, run.summary.get("val_loss", None))
```

---

### Workflow 2: Paper Writing

```
/paper-writing "results/" — venue: NeurIPS
```

Generates LaTeX paper from experiment results. Anti-hallucination enforced: every citation verified via DBLP → CrossRef → `[VERIFY]` tag if unconfirmed.

**Venue templates available:** ICML, NeurIPS, ICLR, CVPR, ACL, AAAI, ACM MM

---

### Workflow 3: Auto Review Loop

```
/auto-review "paper.pdf"
```

The core ARIS loop:
1. Claude Code reads the paper
2. GPT-5.4 reviews as adversarial critic
3. Claude Code rewrites based on critique
4. Score tracked across rounds (target: 8/10 "clear accept")
5. Loop repeats until convergence or max rounds

```
Score progression: 5.2 → 6.1 → 7.3 → 8.0 ✓
```

---

### Workflow 4: Rebuttal

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

**Parameters:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | Target venue |
| `character limit` | required | Hard limit for submission |
| `quick mode` | `false` | Stop after parsing + strategy (no draft) |
| `auto experiment` | `false` | Auto-run supplementary experiments |
| `max stress test rounds` | `1` | GPT-5.4 stress-test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up limit |

**Three safety gates** (rebuttal won't finalize if any fails):
- 🔒 No fabrication — every claim maps to paper/review/user-confirmed result
- 🔒 No overpromise — every promise is user-approved
- 🔒 Full coverage — every reviewer concern is tracked

**Outputs:**
- `PASTE_READY.txt` — exact char count, paste directly to venue
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

---

### Bonus: Slides and Poster

```bash
# Conference presentation
/paper-slides "paper/"     # → Beamer PDF + PPTX + speaker notes + Q&A prep

# Conference poster
/paper-poster "paper/"     # → A0/A1 poster PDF + editable PPTX + SVG
```

---

## Standalone Skills

These skills can be invoked independently or are integrated into the core workflows:

| Skill | Command | Description |
|-------|---------|-------------|
| Research Refine | `/research-refine` | Turn vague ideas into anchored proposals |
| Experiment Plan | `/experiment-plan` | Claim-driven experiment roadmaps |
| Training Check | `/training-check` | Validate training runs before full launch |
| Result to Claim | `/result-to-claim` | Convert raw results to paper claims |
| Ablation Planner | `/ablation-planner` | Design ablation study structure |
| Formula Derivation | `/formula-derivation` | Research formula development and verification |
| Grant Proposal | `/grant-proposal` | Write grant proposals from research |
| Paper Illustration | `/paper-illustration` | Generate figures (Gemini-powered) |
| Citation Claw | `/citation-claw` | Verify and format citations |

---

## Session Recovery & Compact Mode

For short-context models or after interruption:

```
/research-pipeline "topic" — compact: true
```

Generates lean summary files at each checkpoint. Resume after interruption:

```
/research-refine — resume: true
```

ARIS auto-checkpoints the `research-refine` workflow and resumes from last completed phase.

---

## Codex CLI Native Skills

Full skill set available for OpenAI Codex without Claude Code:

```bash
cd skills/skills-codex/
codex "run idea-discovery on discrete diffusion"
```

---

## MCP Server: llm-chat

The `llm-chat` MCP server bridges any OpenAI-compatible API as a reviewer. Start it manually for debugging:

```bash
cd mcp-servers/llm-chat/
node index.js
```

Environment variables:
```bash
export LLM_BASE_URL=https://api.openai.com/v1   # Any OpenAI-compatible endpoint
export LLM_API_KEY=$OPENAI_API_KEY
export LLM_MODEL=gpt-4o                          # Any model name
```

---

## Free Tier via ModelScope

Zero-cost option — no API key required:

```bash
# See full guide: docs/MODELSCOPE_GUIDE.md
export MODELSCOPE_API_KEY=your_modelscope_token
export LLM_BASE_URL=https://api-inference.modelscope.cn/v1
export LLM_MODEL=Qwen/Qwen2.5-72B-Instruct
```

---

## Input Templates

Templates for every workflow live in `templates/`:

```bash
ls templates/
# idea-discovery.md
# experiment-bridge.md
# paper-writing.md
# auto-review.md
# rebuttal.md
# research-refine.md
```

Use them to structure your inputs:

```bash
cat templates/rebuttal.md
# Fill in: paper path, review text, venue, character limit
# Then: /rebuttal [filled template]
```

---

## Directory Structure

```
Auto-claude-code-research-in-sleep/
├── skills/
│   ├── claude-code/          # Claude Code SKILL.md files
│   ├── skills-codex/         # Codex CLI native skills
│   ├── idea-discovery/
│   ├── experiment-bridge/
│   ├── paper-writing/
│   ├── auto-review/
│   ├── rebuttal/             SKILL.md  ← each is a single readable file
│   ├── paper-slides/
│   ├── paper-poster/
│   ├── research-refine/
│   ├── formula-derivation/
│   └── ...
├── mcp-servers/
│   └── llm-chat/             # Universal reviewer bridge
├── templates/                # Input templates for every workflow
├── docs/
│   ├── CURSOR_ADAPTATION.md
│   ├── TRAE_ARIS_RUNBOOK_EN.md
│   ├── ANTIGRAVITY_ADAPTATION.md
│   ├── MODELSCOPE_GUIDE.md
│   ├── MiniMax-GLM-Configuration.md
│   └── CODEX_GEMINI_REVIEW_GUIDE.md
└── README.md
```

---

## Troubleshooting

**Cross-model review not triggering:**
- Check MCP server is running: `codex mcp` or `node mcp-servers/llm-chat/index.js`
- Verify `OPENAI_API_KEY` or `LLM_API_KEY` is set
- Check Claude Code MCP config in `~/.claude/settings.json`

**W&B metrics not loading:**
```python
import wandb
# Ensure you're logged in
wandb.login(key=os.environ["WANDB_API_KEY"])
api = wandb.Api()
# Use full entity/project path
runs = api.runs("your-entity/your-project")
```

**Context window exceeded mid-workflow:**
```
/research-pipeline "topic" — compact: true
```
Then resume with `— resume: true` on the next interrupted skill.

**Citation hallucination warnings (`[VERIFY]` tags):**
These are intentional — ARIS flags unverified citations rather than silently hallucinating. Manually verify flagged citations before submission.

**Rebuttal exceeds character limit:**
Increase `max stress test rounds` — each round trims the draft:
```
/rebuttal "paper/ + reviews" — character limit: 5000, max stress test rounds: 3
```

**ModelScope free tier rate limits:**
Add delay between skill calls or switch to a paid endpoint for overnight runs.

---

## Why Two Models (Not One, Not Four)

- **1 model self-reviewing** → local minima, blind spots (stochastic bandit)
- **2 models cross-reviewing** → adversarial critique breaks blind spots (adversarial bandit)
- **4+ models** → diminishing returns, 2-4× API cost, coordination overhead

Claude Code = fast fluid execution. GPT-5.4/Gemini/GLM = slower, more deliberate critique. Speed × Rigor = better outcomes than either model alone.

---

## Community & Citation

```bibtex
@software{aris2026,
  title  = {ARIS: Auto-Research-In-Sleep},
  author = {wanshuiyin},
  year   = {2026},
  url    = {https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep}
}
```

Join the community: [GitHub Discussions](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/discussions)

Papers accepted using ARIS: CS Conference (8/10 "clear accept"), AAAI 2026 Main Technical (7/10 "good paper, accept").
```
