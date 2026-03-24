```markdown
---
name: aris-autonomous-ml-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only autonomous ML research workflows for Claude Code and other LLM agents, enabling cross-model paper review loops, idea discovery, experiment automation, and paper writing/rebuttal.
triggers:
  - run autonomous ml research with aris
  - set up aris research pipeline
  - use claude code for research while I sleep
  - automate paper review with aris
  - run cross-model research loop
  - generate research ideas with aris
  - set up aris skills for claude code
  - help me use aris for experiment automation
---

# ARIS — Autonomous ML Research In Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous ML research system. It orchestrates cross-model collaboration — one LLM drives execution (Claude Code, Codex) while another acts as a rigorous critic (GPT-5.4, Gemini, GLM, etc.) to break self-play blind spots. The entire system lives in plain `SKILL.md` files that any LLM agent can read and follow.

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### 2. Install skills into Claude Code

Copy or symlink the `skills/` directory into your Claude Code custom skills path:

```bash
# Option A: Copy all skills
cp -r skills/ ~/.claude/skills/

# Option B: Symlink (keeps skills auto-updated)
ln -s $(pwd)/skills ~/.claude/skills/aris
```

### 3. Configure the reviewer MCP server

ARIS uses the `llm-chat` MCP server to call the cross-model reviewer. Set your reviewer API key:

```bash
# For OpenAI (GPT-5.4 xhigh as reviewer)
export OPENAI_API_KEY="your-openai-api-key"

# For any OpenAI-compatible endpoint (GLM, MiniMax, Kimi, DeepSeek, etc.)
export REVIEWER_API_KEY="your-reviewer-api-key"
export REVIEWER_BASE_URL="https://your-provider.com/v1"
export REVIEWER_MODEL="glm-5"  # or minimax-m2.7, kimi-k2.5, etc.
```

Register the MCP server in your Claude Code config (`~/.claude/config.json`):

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "node",
      "args": ["path/to/aris/mcp-servers/llm-chat/index.js"],
      "env": {
        "OPENAI_API_KEY": "${OPENAI_API_KEY}",
        "REVIEWER_BASE_URL": "${REVIEWER_BASE_URL}",
        "REVIEWER_MODEL": "${REVIEWER_MODEL}"
      }
    }
  }
}
```

### 4. (Optional) Codex MCP for OpenAI as reviewer

```bash
npm install -g @openai/codex-mcp
```

Then add to your MCP config:

```json
{
  "mcpServers": {
    "codex": {
      "command": "codex-mcp",
      "env": { "OPENAI_API_KEY": "${OPENAI_API_KEY}" }
    }
  }
}
```

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Skills** | Plain `SKILL.md` files in `skills/`. Claude Code reads them as instructions. |
| **Workflows** | Numbered pipelines (0→4). Each maps to a slash command. |
| **Cross-model review** | Executor LLM writes/runs; reviewer LLM critiques via MCP. Breaks self-play. |
| **Zero lock-in** | Swap Claude for Codex, GPT for GLM — workflows are model-agnostic. |

---

## Workflows & Commands

### Workflow 0 — Literature Survey

```
/literature-survey "discrete diffusion language models"
```

Fetches papers from arXiv/Semantic Scholar, summarizes contributions, builds a gap analysis.

### Workflow 1 — Idea Discovery

```
/idea-discovery "factorized gap in discrete diffusion LMs"
```

Generates ranked research ideas grounded in the literature survey. Integrates `research-refine` and `experiment-plan` for claim-driven proposals.

### Workflow 1.5 — Experiment Bridge

```
/experiment-bridge "run ablation on learning rate schedules"
```

Writes experiment code → GPT cross-model code review → executes on GPU → parses results → saves structured output.

Parameters:
- `code review: true` (default) — GPT reviews code before running
- `compact: true` — generate lean summaries for short-context models

### Workflow 2 — Paper Writing

```
/paper-writing "results/ + idea.md" — venue: NeurIPS
```

Reads your results and idea, writes a full LaTeX paper. Anti-hallucination: all citations verified via DBLP → CrossRef → `[VERIFY]` tags.

Supported venue templates: `CVPR`, `NeurIPS`, `ICLR`, `ICML`, `ACL`, `AAAI`, `ACM MM`

### Workflow 3 — Auto Review Loop

```
/auto-review "paper/"
```

Claude Code submits to GPT reviewer → scores the paper → identifies weaknesses → rewrites → re-scores. Repeats until score converges or max rounds reached.

### Workflow 4 — Rebuttal

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

Parse reviews → atomize concerns → build strategy → draft rebuttal → safety-gate check → GPT stress-test → finalize.

**Safety gates (rebuttal will NOT finalize if any fails):**
- 🔒 No fabrication — every claim maps to paper/review/user-confirmed result
- 🔒 No overpromise — every promise is user-approved
- 🔒 Full coverage — every reviewer concern addressed

**Output files:**
- `PASTE_READY.txt` — exact char count, ready to paste to venue portal
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | ICML / NeurIPS / ICLR / CVPR / ACL / AAAI / ACM |
| `character limit` | required | Hard char cap for rebuttal text |
| `quick mode` | `false` | Stop after parse + strategy (no draft) |
| `auto experiment` | `false` | Auto-run supplementary experiments when reviewers ask |
| `max stress test rounds` | `1` | GPT-5.4 xhigh adversarial stress-test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up round limit |

### Full Pipeline (end-to-end)

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

Chains Workflows 0 → 1 → 1.5 → 2 → 3 automatically.

**Targeted mode** (improve a specific paper using its codebase):

```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

Mix and match:
- `ref paper` only → "what can be improved here?"
- `base repo` only → "what can I build with this code?"
- both → "fix the paper's weaknesses using this code"

---

## Presentation Outputs

```bash
# Conference slides (Beamer PDF + PPTX + speaker notes + Q&A prep)
/paper-slides "paper/"

# Conference poster (A0/A1 PDF + PPTX + SVG, venue colors)
/paper-poster "paper/"
```

---

## Standalone Utility Skills

```bash
# Verify training is progressing correctly
/training-check "logs/"

# Convert raw results to paper-ready claims
/result-to-claim "results/metrics.json"

# Plan ablation study from existing results
/ablation-planner "results/ + method.md"

# Derive and verify research formulas
/formula-derivation "method description"

# Generate grant proposal
/grant-proposal "research direction"

# Generate paper illustrations (uses Gemini)
/paper-illustration "paper/figures/"
```

---

## Alternative Model Combinations (No Claude/OpenAI Required)

ARIS works with any OpenAI-compatible API. Configure via the `llm-chat` MCP server:

```python
# mcp-servers/llm-chat/config.py pattern
import os

REVIEWER_CONFIG = {
    "base_url": os.environ["REVIEWER_BASE_URL"],
    "api_key": os.environ["REVIEWER_API_KEY"],
    "model": os.environ.get("REVIEWER_MODEL", "glm-5"),
}
```

Tested combinations:

| Executor | Reviewer | Config guide |
|----------|----------|-------------|
| Claude Code | GPT-5.4 xhigh | Default setup |
| Claude Code | GLM-5 | Set `REVIEWER_MODEL=glm-5` |
| Claude Code | MiniMax-M2.7 | `docs/MiniMax-GLM-Configuration.md` |
| Codex CLI | Gemini | `docs/CODEX_GEMINI_REVIEW_GUIDE.md` |
| Claude Code | Kimi-K2.5 | Set `REVIEWER_MODEL=kimi-k2.5` |
| Claude Code | DeepSeek | Set `REVIEWER_MODEL=deepseek-r1` |

**Free tier:** Use ModelScope endpoints — see `docs/MODELSCOPE_GUIDE.md`.

---

## Using ARIS in Other IDEs

| IDE | Guide |
|-----|-------|
| Cursor | `docs/CURSOR_ADAPTATION.md` |
| Trae (ByteDance) | `docs/TRAE_ARIS_RUNBOOK_EN.md` |
| Antigravity (Google) | `docs/ANTIGRAVITY_ADAPTATION.md` |
| Codex CLI | `skills/skills-codex/` |
| Windsurf | Same as Cursor — copy skills, register MCP |

For **Codex CLI** specifically:

```bash
# Install Codex CLI
npm install -g @openai/codex

# Use ARIS Codex-native skills
codex --skills path/to/aris/skills/skills-codex/ "run research pipeline on X"
```

---

## Session Recovery & Compact Mode

For long research sessions that may be interrupted:

```
/research-pipeline "topic" — compact: true
```

This generates lean `*_compact.md` summary files at each checkpoint. To resume after interruption:

```
/research-pipeline resume — checkpoint: workflow1_checkpoint.md
```

The `research-refine` skill also auto-saves checkpoints and can resume mid-workflow.

---

## Input Templates

Pre-filled templates for every workflow are in `templates/`:

```bash
ls templates/
# research-pipeline.md
# paper-writing.md
# rebuttal.md
# experiment-bridge.md
# paper-slides.md
# ...
```

Use a template as your starting prompt:

```bash
cat templates/rebuttal.md
# Fill in the fields, then paste into Claude Code
```

---

## Directory Structure

```
Auto-claude-code-research-in-sleep/
├── skills/                    # Core SKILL.md files (install these)
│   ├── research-pipeline/SKILL.md
│   ├── literature-survey/SKILL.md
│   ├── idea-discovery/SKILL.md
│   ├── experiment-bridge/SKILL.md
│   ├── paper-writing/SKILL.md
│   ├── auto-review/SKILL.md
│   ├── rebuttal/SKILL.md
│   ├── paper-slides/SKILL.md
│   ├── paper-poster/SKILL.md
│   ├── training-check/SKILL.md
│   ├── result-to-claim/SKILL.md
│   ├── ablation-planner/SKILL.md
│   ├── formula-derivation/SKILL.md
│   ├── skills-codex/          # Codex CLI native versions
│   └── ...
├── mcp-servers/
│   └── llm-chat/              # OpenAI-compatible reviewer bridge
├── templates/                 # Input templates for each workflow
├── docs/                      # Adaptation guides per IDE/model
└── README.md
```

---

## Common Patterns

### Pattern 1: Overnight research run

```bash
# Start before bed — wake up to a draft paper
/research-pipeline "length generalization in transformers" \
  — venue: NeurIPS \
  — base repo: https://github.com/EleutherAI/gpt-neox
```

### Pattern 2: Improve an existing paper

```bash
# Point at a paper + its codebase
/research-pipeline "improve positional encoding" \
  — ref paper: https://arxiv.org/abs/2104.09864 \
  — base repo: https://github.com/google-research/t5x
```

### Pattern 3: Reviews just dropped — fast rebuttal

```bash
# quick mode = just parse and strategize, no draft yet
/rebuttal "paper/ + reviews/" \
  — venue: ICML \
  — character limit: 5000 \
  — quick mode: true
```

Review the strategy, then run again without `quick mode` to draft.

### Pattern 4: Supplementary experiments during rebuttal

```bash
/rebuttal "paper/ + reviews/" \
  — venue: NeurIPS \
  — character limit: 5000 \
  — auto experiment: true   # triggers /experiment-bridge for new ablations
```

### Pattern 5: Free tier with ModelScope

```bash
export REVIEWER_BASE_URL="https://api-inference.modelscope.cn/v1"
export REVIEWER_API_KEY="${MODELSCOPE_API_KEY}"
export REVIEWER_MODEL="Qwen/Qwen2.5-72B-Instruct"
```

---

## Troubleshooting

### MCP server not found

```bash
# Check MCP server is registered
claude mcp list

# Re-register manually
claude mcp add llm-chat node path/to/aris/mcp-servers/llm-chat/index.js
```

### Reviewer returning empty responses

Check your reviewer endpoint and model name:

```bash
# Test the llm-chat MCP directly
curl "${REVIEWER_BASE_URL}/chat/completions" \
  -H "Authorization: Bearer ${REVIEWER_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"'"${REVIEWER_MODEL}"'","messages":[{"role":"user","content":"ping"}]}'
```

### Citations marked [VERIFY]

This is intentional anti-hallucination behavior. ARIS flags citations it couldn't confirm via DBLP/CrossRef. Manually verify these before submission — do not remove the tag without checking.

### Session context overflow

Use compact mode to reduce token usage:

```
/research-pipeline "topic" — compact: true
```

Or resume from a checkpoint:

```
/auto-review resume — checkpoint: review_round_3.md
```

### Rebuttal exceeds character limit

The safety gate will block finalization. Add `max stress test rounds: 2` to force more aggressive compression, or manually edit `REBUTTAL_DRAFT_rich.md` and re-run the finalization step.

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `skills/research-pipeline/SKILL.md` | Master end-to-end pipeline |
| `skills/rebuttal/SKILL.md` | Post-submission rebuttal (Workflow 4) |
| `skills/experiment-bridge/SKILL.md` | Code → review → GPU run → parse results |
| `skills/auto-review/SKILL.md` | Cross-model iterative review loop |
| `mcp-servers/llm-chat/` | Reviewer MCP bridge (any OpenAI-compatible API) |
| `docs/MiniMax-GLM-Configuration.md` | MiniMax + GLM dual-model setup |
| `docs/MODELSCOPE_GUIDE.md` | Free tier setup |
| `docs/CURSOR_ADAPTATION.md` | Cursor IDE setup |
| `templates/` | Ready-to-fill input templates |
```
