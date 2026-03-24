```markdown
---
name: aris-autonomous-ml-research
description: Skill for using ARIS (Auto-Research-In-Sleep) — a Markdown-only autonomous ML research workflow system that orchestrates cross-model LLM collaboration for paper review, idea generation, experiment automation, and research writing.
triggers:
  - "set up ARIS for autonomous research"
  - "run research pipeline while I sleep"
  - "automate ML paper writing with Claude Code"
  - "cross-model LLM research review loop"
  - "use ARIS to generate research ideas"
  - "run experiment bridge with ARIS"
  - "write rebuttal with ARIS"
  - "autonomous research workflow with Claude Code skills"
---

# ARIS — Autonomous ML Research In Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS is a **zero-dependency, Markdown-only** autonomous ML research workflow system. There is no framework to install, no database, no daemon — every "skill" is a plain `SKILL.md` file that any LLM agent can read and execute. ARIS orchestrates **cross-model collaboration**: one model (e.g. Claude Code) executes research tasks while another (e.g. Codex / GPT-5.4) acts as a rigorous adversarial reviewer, catching blind spots that self-review misses.

Works with: **Claude Code**, Codex CLI, Cursor, Trae, Antigravity, Windsurf, or any LLM agent.

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### 2. Install Skills into Claude Code

Copy the skills directory so Claude Code can discover them:

```bash
# Install all core skills globally
cp -r skills/ ~/.claude/skills/

# Or symlink for live updates
ln -s $(pwd)/skills ~/.claude/skills/aris
```

For **Codex CLI**, skills live under `skills/skills-codex/`:

```bash
cp -r skills/skills-codex/ ~/.codex/skills/
```

### 3. Configure the Cross-Model Reviewer (MCP)

ARIS uses an MCP server to call the reviewer model. Add to your `claude_desktop_config.json` or `~/.claude/config.json`:

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

For **alternative reviewers** (no OpenAI required), use the bundled `llm-chat` MCP server:

```bash
cd mcp-servers/llm-chat
npm install
```

Then configure your preferred OpenAI-compatible endpoint:

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "node",
      "args": ["mcp-servers/llm-chat/index.js"],
      "env": {
        "LLM_API_KEY": "$YOUR_API_KEY",
        "LLM_BASE_URL": "https://api.your-provider.com/v1",
        "LLM_MODEL": "your-model-name"
      }
    }
  }
}
```

Tested reviewer models: `GLM-5`, `MiniMax-M2.7`, `Kimi-K2.5`, `LongCat`, `DeepSeek`.

---

## Core Workflows

ARIS has four main workflows, each invoked by a slash command in Claude Code.

### Workflow 0 — Full Pipeline (Idea → Paper)

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

With a reference paper and base repo:

```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ref paper` | — | ArXiv URL — ARIS reads weaknesses, generates ideas that fix them |
| `base repo` | — | GitHub URL — cloned as starting codebase |
| `compact` | `false` | Generate lean summary files for short-context models |

### Workflow 1 — Idea Discovery

```
/idea-discovery "sparse attention in long-context transformers"
```

Generates novel research ideas, searches related work via DBLP/CrossRef, scores novelty, and produces a ranked idea list. Integrates `research-refine` and `experiment-plan` automatically.

### Workflow 1.5 — Experiment Bridge

```
/experiment-bridge "train baseline on CIFAR-10 with ResNet-50"
```

With cross-model code review before GPU deployment:

```
/experiment-bridge "run ablation study" — code review: true, base repo: https://github.com/org/project
```

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `code review` | `true` | GPT-5.4 reviews code before GPU deployment |
| `base repo` | — | Clone and use as experiment base |
| `compact` | `false` | Compact output for short-context sessions |

### Workflow 2 — Paper Writing

```
/paper-writing "results/ + method description"
```

Writes a full LaTeX paper draft in your target venue format. Enforces citation hygiene: DBLP → CrossRef → `[VERIFY]` — no hallucinated references.

Venue templates available: CVPR, NeurIPS, ICML, ICLR, ACL, AAAI, ACM MM.

### Workflow 3 — Auto Review Loop

```
/auto-review "paper/"
```

Claude Code submits the paper to the reviewer model, collects structured critique, patches weaknesses, and re-submits. Repeats until the score plateaus or target is reached.

```
/auto-review "paper/" — target score: 8, max rounds: 5
```

Watch the score curve rise with each round (like the `auto_review_score_curve.png` in the repo).

### Workflow 4 — Rebuttal

```
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000
```

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `venue` | `ICML` | Target venue: ICML/NeurIPS/ICLR/CVPR/ACL/AAAI/ACM |
| `character limit` | **Required** | Hard character limit for rebuttal text |
| `quick mode` | `false` | Stop after parsing + strategy (Phase 0-3) |
| `auto experiment` | `false` | Auto-run supplementary experiments for new evidence requests |
| `max stress test rounds` | `1` | GPT-5.4 xhigh stress-test iterations |
| `max followup rounds` | `3` | Per-reviewer follow-up round limit |

Outputs:
- `PASTE_READY.txt` — exact char count, ready to paste to venue portal
- `REBUTTAL_DRAFT_rich.md` — extended version for manual editing

Three safety gates (rebuttal will NOT finalize if any fail):
- 🔒 **No fabrication** — every claim maps to paper/review/confirmed result
- 🔒 **No overpromise** — every promise is user-approved
- 🔒 **Full coverage** — every reviewer concern is tracked

---

## Standalone Skills

These skills can be used independently or are integrated into the main workflows:

```
/research-refine "vague idea about attention mechanisms"
/experiment-plan "proposed method X with hypothesis Y"
/training-check "training run logs/"
/result-to-claim "results/ table"
/ablation-planner "method with components A, B, C"
/paper-slides "paper/"          # → Beamer PDF + PPTX + speaker notes
/paper-poster "paper/"          # → A0/A1 poster PDF + PPTX + SVG
/formula-derivation "derive ELBO for VAE with discrete latents"
/grant-proposal "research direction + funding body"
/paper-illustration "figure description"   # Uses Gemini for visuals
```

---

## Alternative Model Combinations

ARIS does not require Claude or OpenAI. Any two OpenAI-compatible models work:

### MiniMax + GLM Configuration

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "node",
      "args": ["mcp-servers/llm-chat/index.js"],
      "env": {
        "LLM_API_KEY": "$MINIMAX_API_KEY",
        "LLM_BASE_URL": "https://api.minimax.chat/v1",
        "LLM_MODEL": "MiniMax-M2.7"
      }
    }
  }
}
```

See full guide: `docs/MiniMax-GLM-Configuration.md`

### Free Tier via ModelScope

Zero-cost setup using ModelScope-hosted models:

```bash
# See docs/MODELSCOPE_GUIDE.md for full setup
export MODELSCOPE_API_KEY=$MODELSCOPE_API_KEY
```

### Codex + Gemini Review

Codex executes, Gemini reviews via local MCP bridge:

```bash
# See docs/CODEX_GEMINI_REVIEW_GUIDE.md
cd mcp-servers/gemini-review
npm install
export GEMINI_API_KEY=$GEMINI_API_KEY
```

---

## Real Usage Examples

### Example 1: End-to-End Research from a Reference Paper

```
# In Claude Code chat:
/research-pipeline "improve the masked diffusion model" \
  — ref paper: https://arxiv.org/abs/2406.04329 \
  — base repo: https://github.com/kuleshov-group/mdlm \
  — compact: true
```

ARIS will:
1. Fetch and parse the reference paper
2. Identify weaknesses and gaps
3. Clone the base repo
4. Generate ideas that specifically address those weaknesses using that code
5. Plan and run experiments
6. Write a paper draft

### Example 2: Score Improvement Loop

```
# Review your draft and iterate until score ≥ 7
/auto-review "paper/" — target score: 7, max rounds: 6
```

### Example 3: Rebuttal After Reviews Drop

```
# Reviews just arrived — quick strategy pass first
/rebuttal "paper/ + reviews/" \
  — venue: NeurIPS \
  — character limit: 4000 \
  — quick mode: true

# Review the strategy, then run full rebuttal
/rebuttal "paper/ + reviews/" \
  — venue: NeurIPS \
  — character limit: 4000 \
  — auto experiment: true
```

### Example 4: Standalone Experiment with Code Review

```
/experiment-bridge "ablation: remove component B from model" \
  — code review: true \
  — base repo: https://github.com/org/my-model
```

### Example 5: Conference Presentation Materials

```
# After acceptance
/paper-slides "paper/" 
# Produces: slides.pdf (Beamer), slides.pptx, speaker_notes.md, qa_prep.md

/paper-poster "paper/" — format: A0
# Produces: poster.pdf, poster.pptx, poster.svg
```

---

## Input Templates

Use the bundled templates in `templates/` to structure your inputs:

```bash
ls templates/
# research-pipeline-template.md
# experiment-bridge-template.md
# rebuttal-template.md
# paper-writing-template.md
# ...
```

Copy and fill in a template before invoking a workflow:

```bash
cp templates/research-pipeline-template.md my-research-input.md
# Edit my-research-input.md, then:
# /research-pipeline — input: my-research-input.md
```

---

## Session Recovery and Compact Mode

For long research sessions that may be interrupted:

```
/research-pipeline "my topic" — compact: true
```

Compact mode generates lean `*_summary.md` checkpoint files. On interruption:

```
# Resume from checkpoint
/research-pipeline — resume: true
```

The `research-refine` checkpoint auto-saves state between phases, enabling auto-resume after context window limits or crashes.

---

## IDE-Specific Adaptation Guides

| IDE | Guide |
|-----|-------|
| Cursor | `docs/CURSOR_ADAPTATION.md` |
| Trae (ByteDance) | `docs/TRAE_ARIS_RUNBOOK_EN.md` |
| Antigravity (Google) | `docs/ANTIGRAVITY_ADAPTATION.md` |
| Codex CLI | `skills/skills-codex/` |
| OpenClaw | `docs/OPENCLAW_ADAPTATION.md` |
| Alibaba Coding Plan | `docs/ALI_CODING_PLAN_GUIDE.md` |

---

## Troubleshooting

### MCP server not connecting

```bash
# Verify the MCP server starts cleanly
node mcp-servers/llm-chat/index.js
# Should print: MCP server listening...

# Check environment variables are exported
echo $OPENAI_API_KEY
echo $LLM_API_KEY
```

### Reviewer model not responding

Check your `LLM_BASE_URL` ends with `/v1` and the model name matches exactly what the provider expects. Test with a direct curl:

```bash
curl "$LLM_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $LLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "your-model-name", "messages": [{"role": "user", "content": "ping"}]}'
```

### Citation hallucination warnings

Workflow 2 enforces DBLP → CrossRef → `[VERIFY]` chain. If you see `[VERIFY]` tags in your draft, those citations need manual confirmation before submission. This is intentional anti-hallucination behavior.

### Context window exceeded during long runs

Enable compact mode to generate lean checkpoints:

```
/research-pipeline "topic" — compact: true
```

Then resume after starting a fresh session:

```
/research-pipeline — resume: true
```

### Score not improving after several rounds

The reviewer may be saturated on the current draft structure. Try:

```
/auto-review "paper/" — max rounds: 3, fresh reviewer: true
```

Or manually intervene between rounds:

```
/auto-review "paper/" — interactive: true
```

---

## Project Structure

```
Auto-claude-code-research-in-sleep/
├── skills/                        # Core SKILL.md files for Claude Code
│   ├── research-pipeline/SKILL.md
│   ├── idea-discovery/SKILL.md
│   ├── experiment-bridge/SKILL.md
│   ├── paper-writing/SKILL.md
│   ├── auto-review/SKILL.md
│   ├── rebuttal/SKILL.md
│   ├── paper-slides/SKILL.md
│   ├── paper-poster/SKILL.md
│   ├── research-refine/SKILL.md
│   ├── experiment-plan/SKILL.md
│   ├── training-check/SKILL.md
│   ├── result-to-claim/SKILL.md
│   ├── ablation-planner/SKILL.md
│   ├── formula-derivation/SKILL.md
│   ├── grant-proposal/SKILL.md
│   ├── paper-illustration/SKILL.md
│   └── skills-codex/              # Codex CLI variants
├── mcp-servers/
│   ├── llm-chat/                  # Universal OpenAI-compatible reviewer MCP
│   └── gemini-review/             # Gemini-specific review bridge
├── templates/                     # Input templates for each workflow
├── docs/                          # IDE adaptation guides, configuration docs
└── README.md
```

---

## Key Design Principles

1. **Zero dependencies** — every skill is a plain Markdown file; no pip install, no npm, no Docker
2. **No lock-in** — swap any model at any layer; the workflow is the value, not the platform
3. **Cross-model adversarial review** — two models break self-play blind spots that single-model review cannot catch
4. **Checkpointed pipelines** — long research workflows survive interruption via compact mode and auto-resume
5. **Anti-hallucination by design** — citation chains, safety gates on rebuttals, and `[VERIFY]` markers enforce factual grounding
```
