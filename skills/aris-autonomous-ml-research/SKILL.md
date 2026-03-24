```markdown
---
name: aris-autonomous-ml-research
description: ARIS (Auto-Research-In-Sleep) — Markdown-only autonomous ML research workflows using cross-model review loops, idea discovery, experiment automation, and paper writing with Claude Code or any LLM agent.
triggers:
  - set up ARIS for autonomous research
  - run research pipeline while I sleep
  - use ARIS to review my paper
  - automate ML experiments with Claude Code
  - cross-model paper review loop
  - generate research ideas with ARIS
  - write paper rebuttal automatically
  - install ARIS skills for Claude Code
---

# ARIS — Autonomous ML Research in Sleep

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

ARIS (Auto-Research-In-Sleep) is a **zero-dependency, Markdown-only** autonomous ML research system. There is no framework, no daemon, no database — every skill is a plain `SKILL.md` file that any LLM agent can read and execute. ARIS orchestrates cross-model collaboration: one model (Claude Code, Codex, etc.) executes research while a second model acts as critical reviewer, breaking self-play blind spots.

**Core capabilities:**
- Autonomous idea discovery and literature gap analysis
- Cross-model adversarial review loops (Claude ↔ GPT / Kimi / GLM / DeepSeek)
- Automated experiment planning and execution
- Full paper writing (LaTeX) with venue templates
- Post-submission rebuttal drafting with safety gates
- Conference slides, posters, and grant proposals

---

## Installation

### Prerequisites

```bash
# Claude Code CLI (primary supported agent)
npm install -g @anthropic-ai/claude-code

# Optional: Codex CLI (for OpenAI-side reviewer)
npm install -g @openai/codex
```

### Clone ARIS

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### Install Skills into Claude Code

Copy skills to Claude Code's custom skills directory:

```bash
# Install all skills at once
cp -r skills/*/SKILL.md ~/.claude/skills/

# Or install specific skills
cp skills/research-pipeline/SKILL.md ~/.claude/skills/research-pipeline.md
cp skills/rebuttal/SKILL.md ~/.claude/skills/rebuttal.md
cp skills/experiment-bridge/SKILL.md ~/.claude/skills/experiment-bridge.md
```

### Configure Cross-Model Reviewer (MCP)

ARIS uses a Codex MCP server so Claude Code can call GPT as reviewer. Add to your `~/.claude/mcp_config.json`:

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

For the **alternative `llm-chat` MCP** (Kimi, GLM, MiniMax, DeepSeek — no OpenAI needed):

```json
{
  "mcpServers": {
    "llm-chat": {
      "command": "python",
      "args": ["mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "$YOUR_PROVIDER_API_KEY",
        "LLM_BASE_URL": "https://api.moonshot.cn/v1",
        "LLM_MODEL": "moonshot-v1-128k"
      }
    }
  }
}
```

### Environment Variables

```bash
# Primary executor model (Claude Code reads these automatically)
export ANTHROPIC_API_KEY=...

# Reviewer model — choose ONE of:
export OPENAI_API_KEY=...           # GPT via Codex MCP
export MOONSHOT_API_KEY=...         # Kimi
export ZHIPUAI_API_KEY=...          # GLM-5
export MINIMAX_API_KEY=...          # MiniMax
export DEEPSEEK_API_KEY=...         # DeepSeek
```

---

## Quick Start Commands

All commands are invoked inside Claude Code chat.

### Full Autonomous Pipeline (idea → paper)

```
/research-pipeline "factorized gap in discrete diffusion LMs"
```

With a reference paper and base codebase:

```
/research-pipeline "improve method X" — ref paper: https://arxiv.org/abs/2406.04329, base repo: https://github.com/org/project
```

### Individual Workflow Shortcuts

```
# Workflow 0 — Literature survey only
/literature-review "topic"

# Workflow 1 — Idea discovery
/idea-discovery "research direction"

# Workflow 1.5 — Experiment bridge (run GPU experiments)
/experiment-bridge "experiments/" — code review: true

# Workflow 2 — Paper writing
/paper-write "results/"

# Workflow 3 — Paper review loop
/paper-review "paper/"

# Workflow 4 — Post-submission rebuttal
/rebuttal "paper/ + reviews" — venue: ICML, character limit: 5000

# Presentation outputs
/paper-slides "paper/"
/paper-poster "paper/"
```

---

## Workflows in Detail

### Workflow 0: Literature Review

```
/literature-review "masked language models discrete diffusion"
```

Outputs:
- `literature/survey.md` — structured gap analysis
- `literature/references.bib` — verified BibTeX (DBLP → CrossRef → [VERIFY])
- `literature/gap_matrix.md` — opportunity map

Anti-hallucination: all citations go through DBLP → CrossRef → manual `[VERIFY]` tag if unconfirmed.

---

### Workflow 1: Idea Discovery + Research Refinement

```
/idea-discovery "what's missing in current RLHF approaches"
```

Internally chains:
1. `/literature-review` — find gaps
2. `/research-refine` — turn vague gaps into problem-anchored proposals
3. `/experiment-plan` — claim-driven experiment roadmap
4. GPT reviewer scores each idea (1–10) and requests clarifications

Output files:

```
ideas/
  idea_001.md        # Full proposal + claims
  idea_001_score.md  # GPT review + score
  experiment_plan.md # Prioritized experiment list
```

---

### Workflow 1.5: Experiment Bridge

Runs experiments on GPU, with cross-model code review before deployment:

```
/experiment-bridge "experiments/plan.md" — code review: true, gpu: A100
```

Parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `code review` | `true` | GPT reviews training code before launch |
| `gpu` | auto | GPU type hint for SLURM/local dispatch |
| `compact` | `false` | Generate lean summary for short-context models |
| `max retries` | `3` | Auto-retry failed runs |

Built-in helpers integrated into this workflow:
- `/training-check` — validates training loop sanity (loss, gradients, LR)
- `/result-to-claim` — converts raw numbers to paper-ready claims
- `/ablation-planner` — suggests ablation study design

---

### Workflow 2: Paper Writing

```
/paper-write "results/" — venue: NeurIPS, template: neurips2025
```

Available venue templates:

```bash
templates/
  neurips2025.tex
  icml2025.tex
  iclr2025.tex
  cvpr2025.tex
  acl2025.tex
  aaai2026.tex
  acmmm2025.tex
```

Workflow writes sections in order: Abstract → Introduction → Related Work → Method → Experiments → Conclusion, with GPT reviewing each section and Claude revising.

---

### Workflow 3: Paper Review Loop

```
/paper-review "paper/" — rounds: 3, target score: 7
```

Each round:
1. GPT reads full PDF/LaTeX → scores (1–10) + itemized weaknesses
2. Claude revises paper to address weaknesses
3. Score delta logged to `review/score_history.md`

The `docs/auto_review_score_curve.png` in the repo shows a typical trajectory from 4→8 over 5 rounds.

Parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `rounds` | `5` | Max review-revise iterations |
| `target score` | `7` | Stop early if score reached |
| `reviewer model` | Codex MCP | Override via `— reviewer: llm-chat` |
| `compact` | `false` | Compact session summaries |

---

### Workflow 4: Rebuttal

```
/rebuttal "paper/ + reviews/" — venue: ICML, character limit: 5000
```

Phase breakdown:
- **Phase 0** — Parse all reviews, extract concerns
- **Phase 1** — Atomize concerns (one claim per atom)
- **Phase 2** — Map atoms to paper evidence
- **Phase 3** — Rebuttal strategy (which to concede, which to counter)
- **Phase 4** — Draft rebuttal
- **Phase 5** — GPT stress-test the draft
- **Phase 6** — Finalize with 3 safety gates

Three mandatory safety gates (rebuttal will NOT finalize if any fails):

```
🔒 No fabrication   — every claim maps to paper/review/confirmed result
🔒 No overpromise   — every promise is user-approved before inclusion
🔒 Full coverage    — every reviewer concern has a response
```

Output files:

```
rebuttal/
  PASTE_READY.txt           # Exact char count, ready to paste to venue
  REBUTTAL_DRAFT_rich.md    # Extended version for manual editing
  concern_map.md            # Atom → evidence mapping
  strategy.md               # Response strategy per reviewer
```

Additional parameters:

```
— quick mode: true          # Stop after Phase 3 (analysis only, no draft)
— auto experiment: true     # Run /experiment-bridge for missing evidence
— max stress test rounds: 2 # How many GPT stress-test passes
— max followup rounds: 3    # Per-reviewer follow-up limit
```

---

### Presentation Outputs

```
# Conference slides
/paper-slides "paper/" — format: beamer, include: speaker-notes, qa-prep

# Conference poster
/paper-poster "paper/" — size: A0, venue: NeurIPS
```

Slides output: Beamer PDF + PPTX + `speaker_notes.md` + `qa_prep.md`
Poster output: `tcbposter` → A0/A1 PDF + PPTX + SVG (venue color scheme applied)

---

## Alternative Model Combinations

ARIS works without any Claude or OpenAI API. Configure `llm-chat` MCP to mix any two OpenAI-compatible providers:

```python
# mcp-servers/llm-chat/config.py — example configurations

CONFIGS = {
    # MiniMax executes, GLM reviews
    "minimax_glm": {
        "executor": {
            "base_url": "https://api.minimax.chat/v1",
            "model": "MiniMax-M2.7",
            "api_key_env": "MINIMAX_API_KEY",
        },
        "reviewer": {
            "base_url": "https://open.bigmodel.cn/api/paas/v4",
            "model": "glm-5",
            "api_key_env": "ZHIPUAI_API_KEY",
        }
    },
    # Kimi executes, DeepSeek reviews
    "kimi_deepseek": {
        "executor": {
            "base_url": "https://api.moonshot.cn/v1",
            "model": "moonshot-v1-128k",
            "api_key_env": "MOONSHOT_API_KEY",
        },
        "reviewer": {
            "base_url": "https://api.deepseek.com/v1",
            "model": "deepseek-chat",
            "api_key_env": "DEEPSEEK_API_KEY",
        }
    }
}
```

See `docs/MiniMax-GLM-Configuration.md` for full setup walkthrough.

---

## Free Tier: ModelScope

Run ARIS at zero cost via ModelScope's free API tier:

```bash
# Set ModelScope endpoint in llm-chat config
export LLM_BASE_URL="https://api-inference.modelscope.cn/v1"
export LLM_MODEL="Qwen/Qwen2.5-72B-Instruct"
export LLM_API_KEY="$MODELSCOPE_API_KEY"
```

See `docs/MODELSCOPE_GUIDE.md` for model availability and rate limits.

---

## IDE Adaptations

| IDE | Guide |
|-----|-------|
| Claude Code | Native (this skill) |
| Codex CLI | `skills/skills-codex/` |
| Cursor | `docs/CURSOR_ADAPTATION.md` |
| Trae (ByteDance) | `docs/TRAE_ARIS_RUNBOOK_EN.md` |
| Antigravity (Google) | `docs/ANTIGRAVITY_ADAPTATION.md` |
| Windsurf | Drop `SKILL.md` files into rules directory |

---

## Templates

Input templates for every workflow live in `templates/`:

```bash
templates/
  research-pipeline.md     # Full pipeline input template
  idea-discovery.md
  experiment-bridge.md
  paper-write.md
  paper-review.md
  rebuttal.md
  paper-slides.md
  paper-poster.md
```

Use them to avoid forgetting parameters:

```bash
# Copy template, fill it in, then paste into Claude Code
cat templates/rebuttal.md
```

---

## Session Recovery and Compact Mode

For long pipelines that hit context limits:

```
/research-pipeline "topic" — compact: true
```

This generates lean `*_compact.md` summary files at each phase boundary. On interruption:

```
/research-pipeline resume — from: experiment-bridge
```

ARIS reads the compact summaries and continues without re-running completed phases. The `research-refine` workflow also has auto-checkpoint: it writes `checkpoint.json` after each idea refinement round.

---

## Standalone Utility Skills

These can be called independently or are auto-invoked by core workflows:

```
/training-check "experiments/"          # Validate training loop health
/result-to-claim "results/metrics.json" # Convert numbers to paper claims
/ablation-planner "method/"             # Design ablation study
/formula-derivation "method/idea.md"    # Develop + verify math derivations
/literature-review "topic"              # Standalone survey
/research-refine "idea.md"              # Sharpen a vague idea
/experiment-plan "proposal.md"          # Generate experiment roadmap
/grant-proposal "idea.md"               # Draft grant proposal
/paper-illustration "paper/"            # Generate figures (Gemini)
```

---

## File Structure After Full Pipeline Run

```
project/
  literature/
    survey.md
    references.bib
    gap_matrix.md
  ideas/
    idea_001.md
    idea_001_score.md
    experiment_plan.md
  experiments/
    plan.md
    run_001/
      train.py
      results.json
      training_check.md
  paper/
    main.tex
    main.pdf
    figures/
  review/
    round_1_review.md
    round_1_revised.tex
    score_history.md
  rebuttal/
    PASTE_READY.txt
    REBUTTAL_DRAFT_rich.md
  slides/
    slides.pdf
    slides.pptx
    speaker_notes.md
```

---

## Troubleshooting

**MCP server not found**
```bash
# Verify Codex MCP is running
codex mcp --test

# Check Claude Code sees it
claude mcp list
```

**Skill not recognized**
```bash
# Confirm skill files are in place
ls ~/.claude/skills/

# Reload Claude Code session after adding new skills
# (start a new conversation)
```

**Cross-model reviewer silent / no score returned**
- Check `OPENAI_API_KEY` or alternative provider key is exported
- Verify `mcp_config.json` syntax with `python -m json.tool ~/.claude/mcp_config.json`
- For `llm-chat` MCP: confirm `LLM_BASE_URL` ends without trailing slash

**LaTeX compilation fails in paper-write**
```bash
# ARIS requires texlive — install if missing
sudo apt-get install texlive-full   # Linux
brew install --cask mactex          # macOS
```

**Compact mode not resuming correctly**
- Ensure `checkpoint.json` was written (check `experiments/` dir)
- Specify exact phase: `/research-pipeline resume — from: paper-write`

**Citation hallucination**
- All `[VERIFY]`-tagged citations in `references.bib` need manual confirmation
- Run `/literature-review` with `— strict: true` to block unverified citations entirely

---

## Community and Citation

- 💬 Community: see `README.md` for Discord/WeChat links
- 🏆 Papers accepted with ARIS: see `README.md#community-showcase`
- 📖 BibTeX citation: see `README.md#citation`

```bibtex
@software{aris2026,
  title  = {ARIS: Auto-Research-In-Sleep},
  author = {wanshuiyin},
  year   = {2026},
  url    = {https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep}
}
```
```
