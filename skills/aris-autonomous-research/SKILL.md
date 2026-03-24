```markdown
---
name: aris-autonomous-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only autonomous ML research workflows using cross-model review loops, idea discovery, experiment automation, and paper writing with Claude Code or any LLM agent.
triggers:
  - run autonomous research pipeline
  - set up ARIS research workflow
  - use claude code for ML research
  - automate paper writing with AI
  - cross-model research review loop
  - run experiment automation with ARIS
  - install ARIS skills for claude code
  - generate research ideas while sleeping
---

# ARIS — Autonomous Research In Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous ML research system. Each "skill" is a plain `SKILL.md` file that any LLM agent can read and execute. The system orchestrates **cross-model collaboration**: one model executes (Claude Code / Codex) while another critiques (GPT-5.4 / Gemini / GLM / MiniMax), breaking self-review blind spots without any framework or lock-in.

Core capabilities:
- 🔬 **Idea discovery** from a research direction or existing paper
- 🧪 **Experiment automation** with GPU-ready code generation and W&B tracking
- 📝 **Paper writing** (LaTeX, Beamer slides, A0 poster)
- 🔁 **Cross-model review loops** with score progression
- 📬 **Rebuttal drafting** with safety gates (no fabrication, no overpromise, full coverage)

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### 2. Install skills into Claude Code

Copy the skills directory to your project, or symlink it:

```bash
# Option A: copy skills to your project
cp -r skills/ /your/project/.claude/skills/

# Option B: symlink (keeps skills up to date)
ln -s /path/to/Auto-claude-code-research-in-sleep/skills /your/project/.claude/skills
```

Claude Code auto-discovers `SKILL.md` files in `.claude/skills/**`. No registration step needed.

### 3. Configure the MCP reviewer (cross-model review)

ARIS uses the `llm-chat` MCP server so the executor model can call a second model for review. Install it:

```bash
cd mcp-servers/llm-chat
pip install -r requirements.txt   # or: uv pip install -r requirements.txt
```

Add to your `claude_desktop_config.json` (or Claude Code MCP config):

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python",
      "args": ["/path/to/Auto-claude-code-research-in-sleep/mcp-servers/llm-chat/server.py"],
      "env": {
        "OPENAI_API_KEY": "$OPENAI_API_KEY",
        "LLM_MODEL": "gpt-4o"
      }
    }
  }
}
```

> For alternative reviewers (Kimi, GLM, MiniMax, DeepSeek) set `LLM_BASE_URL` and `LLM_MODEL` to the provider's OpenAI-compatible endpoint. No Claude or OpenAI API required.

### 4. (Optional) Codex MCP for OpenAI Codex as executor

```json
{
  "mcpServers": {
    "codex": {
      "command": "npx",
      "args": ["@openai/codex-mcp"],
      "env": {
        "OPENAI_API_KEY": "$OPENAI_API_KEY"
      }
    }
  }
}
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OPENAI_API_KEY` | For GPT reviewer | OpenAI API key |
| `ANTHROPIC_API_KEY` | For Claude executor | Anthropic API key |
| `LLM_BASE_URL` | Alternative reviewer | OpenAI-compatible base URL |
| `LLM_MODEL` | Alternative reviewer | Model name at that endpoint |
| `WANDB_API_KEY` | Experiment tracking | Weights & Biases key |

---

## Workflows & Commands

### Full pipeline (idea → paper)

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

With a reference paper and base codebase:

```
/research-pipeline "improve positional encoding in transformers" — ref paper: https://arxiv.org/abs/2104.09864, base repo: https://github.com/facebookresearch/fairseq
```

Parameters:

| Flag | Default | Effect |
|---|---|---|
| `ref paper` | — | ARIS reads this paper, finds weaknesses, targets them |
| `base repo` | — | Clone and use this repo as experiment base |
| `compact: true` | false | Generate lean summary files (good for short-context models) |

---

### Workflow 1 — Idea Discovery

```
/idea-discovery "sparse attention in long-context LLMs"
```

What it does:
1. Searches recent arXiv papers in the direction
2. Identifies open gaps and contradiction clusters
3. Generates 3–5 novel ideas with novelty scores
4. Runs `/research-refine` to sharpen the best idea into a problem statement

---

### Workflow 1.5 — Experiment Bridge

```
/experiment-bridge "idea_proposal.md" — base repo: https://github.com/huggingface/diffusers
```

What it does:
1. Reads the sharpened idea from Workflow 1
2. Generates GPU-ready experiment code
3. Runs **GPT cross-model code review** before deployment (`code review: true` by default)
4. Executes training loop with W&B logging
5. Saves results to `experiment_results/`

Example generated experiment scaffold:

```python
# experiment_results/run_001/train.py  (auto-generated by /experiment-bridge)
import wandb
import torch
from torch.utils.data import DataLoader

wandb.init(
    project=os.environ.get("WANDB_PROJECT", "aris-experiment"),
    config={
        "method": "factorized_discrete_diffusion",
        "lr": 3e-4,
        "epochs": 50,
        "batch_size": 32,
    }
)

for epoch in range(config.epochs):
    for batch in dataloader:
        loss = model(batch)
        wandb.log({"loss": loss.item(), "epoch": epoch})
```

---

### Workflow 2 — Literature Review

```
/literature-review "discrete diffusion language models"
```

Anti-hallucination: ARIS verifies every citation via DBLP → CrossRef → marks unverified as `[VERIFY]`. Never fabricates BibTeX.

---

### Workflow 3 — Paper Writing

```
/paper-write "experiment_results/ + idea_proposal.md" — venue: NeurIPS
```

Supported venue templates: `CVPR`, `NeurIPS`, `ICML`, `ICLR`, `ACL`, `AAAI`, `ACM MM`

---

### Workflow 4 — Rebuttal

```
/rebuttal "paper/ + reviews/" — venue: ICML, character limit: 5000
```

Parameters:

| Parameter | Default | Description |
|---|---|---|
| `venue` | `ICML` | Target conference |
| `character limit` | **Required** | Hard character limit |
| `quick mode` | false | Stop after strategy (Phase 0–3), no draft |
| `auto experiment` | false | Auto-run experiments when reviewers ask for new evidence |
| `max stress test rounds` | 1 | GPT-5.4 stress-test passes on draft |
| `max followup rounds` | 3 | Per-reviewer follow-up round limit |

Three safety gates — rebuttal will NOT finalize if any fails:
- 🔒 No fabrication — every claim maps to paper/review/confirmed result
- 🔒 No overpromise — every promise is user-approved
- 🔒 Full coverage — every reviewer concern is tracked

Outputs:
- `PASTE_READY.txt` — exact character count, ready to paste to venue portal
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

---

### Presentation & Poster

```
/paper-slides "paper/"     # Beamer PDF + PPTX + speaker notes + Q&A prep
/paper-poster "paper/"     # A0/A1 poster PDF + PPTX + SVG (venue colors)
```

---

## Standalone Utility Skills

| Skill | Command | What it does |
|---|---|---|
| `training-check` | `/training-check "train.py"` | Diagnose training instability, NaN, slow convergence |
| `result-to-claim` | `/result-to-claim "results.json"` | Convert raw numbers into paper-ready claims |
| `ablation-planner` | `/ablation-planner "idea.md"` | Design minimal ablation study for a method |
| `experiment-plan` | `/experiment-plan "idea.md"` | Claim-driven experiment roadmap |
| `research-refine` | `/research-refine "idea.md"` | Sharpen vague idea into problem-anchored proposal |
| `formula-derivation` | `/formula-derivation "method.md"` | Develop and verify research formulas |
| `paper-illustration` | `/paper-illustration "paper/"` | Generate figures (Gemini-assisted) |
| `grant-proposal` | `/grant-proposal "idea.md"` | Draft grant proposal from research idea |

---

## Alternative Model Combinations

ARIS requires only an OpenAI-compatible endpoint for the reviewer. Set environment variables:

```bash
# Kimi as reviewer
export LLM_BASE_URL="https://api.moonshot.cn/v1"
export LLM_MODEL="moonshot-v1-128k"
export LLM_API_KEY=$MOONSHOT_API_KEY

# DeepSeek as reviewer
export LLM_BASE_URL="https://api.deepseek.com/v1"
export LLM_MODEL="deepseek-chat"
export LLM_API_KEY=$DEEPSEEK_API_KEY

# MiniMax as reviewer
export LLM_BASE_URL="https://api.minimax.chat/v1"
export LLM_MODEL="abab6.5s-chat"
export LLM_API_KEY=$MINIMAX_API_KEY
```

Then in `mcp-servers/llm-chat/server.py` the `LLM_BASE_URL` env var overrides the OpenAI default. No code changes needed.

---

## Using with Codex CLI (no Claude)

ARIS ships a parallel `skills-codex/` directory with the same workflows adapted for OpenAI Codex CLI:

```bash
# Install Codex CLI
npm install -g @openai/codex

# Run a workflow
codex --skill skills/skills-codex/research-pipeline/SKILL.md \
  "improve contrastive learning in vision transformers"
```

---

## Using with Cursor

1. Open Cursor settings → Rules → paste content of `docs/CURSOR_ADAPTATION.md`
2. Copy `skills/` to `.cursorrules-skills/` in your project
3. In chat: `@research-pipeline "your research direction"`

---

## Using with Trae (ByteDance IDE)

See [`docs/TRAE_ARIS_RUNBOOK_EN.md`](docs/TRAE_ARIS_RUNBOOK_EN.md) for full setup. Trae supports SKILL.md natively via its plugin system.

---

## Input Templates

Pre-filled templates for every workflow live in `templates/`:

```
templates/
  research-pipeline.md      # Full pipeline input
  idea-discovery.md
  experiment-bridge.md
  literature-review.md
  paper-write.md
  rebuttal.md
  paper-slides.md
  paper-poster.md
```

Use a template:

```
/research-pipeline — template: templates/research-pipeline.md
```

---

## Project Structure

```
Auto-claude-code-research-in-sleep/
├── skills/
│   ├── research-pipeline/SKILL.md      # Main orchestration workflow
│   ├── idea-discovery/SKILL.md         # Workflow 1
│   ├── experiment-bridge/SKILL.md      # Workflow 1.5
│   ├── literature-review/SKILL.md      # Workflow 2
│   ├── paper-write/SKILL.md            # Workflow 3
│   ├── rebuttal/SKILL.md               # Workflow 4
│   ├── paper-slides/SKILL.md
│   ├── paper-poster/SKILL.md
│   ├── training-check/SKILL.md
│   ├── result-to-claim/SKILL.md
│   ├── ablation-planner/SKILL.md
│   ├── experiment-plan/SKILL.md
│   ├── research-refine/SKILL.md
│   ├── formula-derivation/SKILL.md
│   └── skills-codex/                   # Codex CLI variants
├── mcp-servers/
│   └── llm-chat/                       # OpenAI-compatible reviewer MCP
├── templates/                          # Input templates per workflow
├── docs/
│   ├── CURSOR_ADAPTATION.md
│   ├── TRAE_ARIS_RUNBOOK_EN.md
│   ├── ANTIGRAVITY_ADAPTATION.md
│   ├── MODELSCOPE_GUIDE.md             # Free tier setup
│   ├── MiniMax-GLM-Configuration.md
│   └── CODEX_GEMINI_REVIEW_GUIDE.md
└── README.md
```

---

## Common Patterns

### Pattern 1: Start from an arXiv paper you want to beat

```
/research-pipeline "improve method" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/repo
```

ARIS reads the paper → identifies weaknesses → clones repo → generates targeted ideas → runs experiments → writes paper.

### Pattern 2: Resume interrupted session

Add `compact: true` to any workflow. ARIS writes a lean `SESSION_SUMMARY.md`. On resume:

```
/research-pipeline — resume: SESSION_SUMMARY.md
```

### Pattern 3: Jump into the middle of a pipeline

Already have results? Jump to paper writing:

```
/paper-write "my_results/ + my_idea.md" — venue: NeurIPS
```

Already have a paper? Jump to rebuttal:

```
/rebuttal "paper/ + reviews/" — venue: ICML, character limit: 5000
```

### Pattern 4: Free tier via ModelScope

```bash
export LLM_BASE_URL="https://api-inference.modelscope.cn/v1"
export LLM_MODEL="Qwen/Qwen2.5-72B-Instruct"
export LLM_API_KEY=$MODELSCOPE_API_KEY
```

See `docs/MODELSCOPE_GUIDE.md` for zero-cost setup.

---

## Troubleshooting

**Skills not discovered by Claude Code**

Ensure SKILL.md files are under `.claude/skills/` relative to your project root. Claude Code scans this path at startup.

**MCP reviewer not connecting**

```bash
# Test the llm-chat server directly
cd mcp-servers/llm-chat
python server.py --test
# Should print: {"status": "ok", "model": "gpt-4o"}
```

**W&B logging not working in experiment-bridge**

```bash
wandb login  # uses WANDB_API_KEY env var, or prompts for manual entry
```

**Citation hallucination in literature-review**

All unverified citations are tagged `[VERIFY]` in output. Search DBLP manually for flagged entries before including in your paper. Never remove the `[VERIFY]` tag without confirming.

**Rebuttal exceeds character limit**

ARIS tracks character count per section. If a draft exceeds the limit, it automatically trims supporting evidence (keeps claims, removes elaboration). You can also pass `quick mode: true` to get the strategy without the draft, then write targeted sections manually.

**Cross-model review loop not running (self-review fallback)**

If the `llm-chat` MCP is unreachable, ARIS falls back to single-model review with a warning in the output. Check MCP server logs:

```bash
tail -f ~/.claude/mcp-logs/llm-chat.log
```

**Session context overflow**

Use `compact: true` on any workflow invocation to produce a compressed `SESSION_SUMMARY.md` that fits in a fresh context window.

---

## Extending ARIS

Every skill is a plain Markdown file. To create a custom skill:

```markdown
# my-custom-skill

## Trigger
When the user says "run my custom analysis"...

## Steps
1. Read input files
2. Call `mcp__llm-chat__chat` with the review prompt
3. Write output to `custom_output/`

## Output
- `custom_output/analysis.md`
- `custom_output/score.json`
```

Save as `.claude/skills/my-custom-skill/SKILL.md` and Claude Code will discover it automatically.
```
