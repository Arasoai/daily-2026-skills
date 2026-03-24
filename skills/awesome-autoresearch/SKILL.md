```markdown
---
name: awesome-autoresearch
description: Curated index of autonomous improvement loops, research agents, and autoresearch-style systems inspired by Karpathy's autoresearch.
triggers:
  - set up an autoresearch loop
  - build a self-improving agent
  - implement autonomous research workflow
  - create an experiment optimization loop
  - add autoresearch skill to my project
  - build a keep-or-revert improvement loop
  - set up a research agent pipeline
  - automate ml experimentation with agents
---

# 🔬 Awesome Autoresearch

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated index of autonomous improvement loops, research agents, and autoresearch-style systems. The core pattern: an LLM agent proposes changes, runs experiments, measures a metric, and keeps or reverts — looping until a budget is exhausted or a threshold is met.

---

## What Is Autoresearch?

Autoresearch (originated by [karpathy/autoresearch](https://github.com/karpathy/autoresearch)) is an **autonomous experiment loop** where:

1. An LLM agent reads a codebase and a goal metric
2. It proposes a targeted change (hypothesis)
3. The change is applied and the metric is measured
4. If the metric improves → keep; otherwise → revert
5. Repeat within a fixed compute/time budget

The pattern generalizes to any measurable objective: model loss, Sharpe ratio, test pass rate, API latency, prompt quality, etc.

---

## Core Loop Pattern

```python
# Canonical keep-or-revert autoresearch loop
import subprocess, shutil, json
from pathlib import Path

METRIC_CMD = ["python", "eval.py"]          # returns JSON {"score": float}
BUDGET = 20                                  # number of iterations
GOAL = "maximize score"

def measure() -> float:
    result = subprocess.run(METRIC_CMD, capture_output=True, text=True)
    return json.loads(result.stdout)["score"]

def run_loop(agent_propose_fn):
    best_score = measure()
    print(f"Baseline: {best_score:.4f}")

    for step in range(BUDGET):
        # Agent proposes a diff/edit
        agent_propose_fn(goal=GOAL, step=step, best=best_score)

        score = measure()
        if score > best_score:
            best_score = score
            print(f"[{step}] ✅ Improved → {score:.4f}")
            # Commit the change (git add -A && git commit)
            subprocess.run(["git", "commit", "-am", f"step {step}: {score:.4f}"])
        else:
            print(f"[{step}] ❌ Reverted  ({score:.4f} < {best_score:.4f})")
            # Revert to last good state
            subprocess.run(["git", "checkout", "--", "."])

    print(f"Final best: {best_score:.4f}")
```

---

## Installation Patterns by Platform

### Claude Code Skill (SKILL.md / CLAUDE.md)

Create `CLAUDE.md` or `.claude/skills/autoresearch.md` in your repo:

```markdown
## Autoresearch Loop

You are running an autonomous improvement loop. Each iteration:
1. Read `GOAL.md` for the objective and metric command
2. Propose ONE focused change to the codebase
3. Apply the change
4. Run: `python eval.py` → parse `{"score": float}`
5. If score improves over baseline: `git commit -am "step N: <score>"`
6. Else: `git checkout -- .`
7. Log to `experiments.jsonl`
8. Repeat until BUDGET iterations or target score reached
```

### GOAL.md Pattern ([jmilinovich/goal-md](https://github.com/jmilinovich/goal-md))

```markdown
# GOAL.md

## Objective
Minimize validation bits-per-byte on the Shakespeare dataset.

## Metric Command
```bash
python eval.py --split val
```
Returns: `{"val_bpb": float}` — lower is better.

## Budget
- Max iterations: 20
- Max wall time: 2 hours

## Constraints
- Single file edits only (`model.py` or `train.py`)
- No external API calls
- Must run on single A100
```

### Codex / OpenAI CLI

```bash
# Install Codex CLI
npm install -g @openai/codex

# Run autoresearch loop via Codex
codex "Read GOAL.md. Run the autoresearch loop: propose a change, measure eval.py output, keep if improved else revert. Repeat 20 times. Log each step to experiments.jsonl."
```

---

## Experiment Logging

```python
# experiments.jsonl writer — append each step
import json, datetime

def log_step(step: int, score: float, baseline: float, diff: str, kept: bool):
    record = {
        "step": step,
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "score": score,
        "baseline": baseline,
        "delta": score - baseline,
        "kept": kept,
        "diff_summary": diff[:200],   # first 200 chars of unified diff
    }
    with open("experiments.jsonl", "a") as f:
        f.write(json.dumps(record) + "\n")
```

---

## Domain-Specific Configurations

### ML Training Loss (original pattern)

```python
# eval.py for language model val_bpb
import torch, json
model = load_checkpoint("ckpt_latest.pt")
val_bpb = evaluate_bpb(model, "data/val.bin")
print(json.dumps({"score": -val_bpb}))  # negate so higher=better
```

### API / Prompt Optimization

```python
# eval.py for prompt quality
import os, json
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

def score_prompt(prompt_file="system_prompt.txt") -> float:
    prompt = open(prompt_file).read()
    scores = []
    for test_case in load_test_cases("test_cases.json"):
        resp = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "system", "content": prompt},
                      {"role": "user", "content": test_case["input"]}]
        )
        scores.append(judge(resp.choices[0].message.content, test_case["expected"]))
    return sum(scores) / len(scores)

print(json.dumps({"score": score_prompt()}))
```

### GPU Kernel Optimization ([RightNow-AI/autokernel](https://github.com/RightNow-AI/autokernel))

```python
# eval.py for kernel throughput
import subprocess, json, re

result = subprocess.run(
    ["python", "benchmark_kernel.py", "--kernel", "attn_fwd"],
    capture_output=True, text=True
)
tflops = float(re.search(r"TFLOP/s: ([\d.]+)", result.stdout).group(1))
print(json.dumps({"score": tflops}))
```

### Trading Strategy ([chrisworsey55/atlas-gic](https://github.com/chrisworsey55/atlas-gic))

```python
# eval.py for Sharpe ratio
import json
from backtest import run_backtest

sharpe = run_backtest(
    strategy_file="strategy.py",
    data_path="data/ohlcv_2023.parquet",
    initial_capital=100_000
)
print(json.dumps({"score": sharpe}))
```

---

## Multi-GPU / Parallel Loops ([iii-hq/n-autoresearch](https://github.com/iii-hq/n-autoresearch))

```python
# parallel_loop.py — run N agents on different hypotheses simultaneously
import asyncio, json
from pathlib import Path

async def run_agent(agent_id: int, gpu_id: int, hypothesis: dict) -> dict:
    env = {"CUDA_VISIBLE_DEVICES": str(gpu_id)}
    proc = await asyncio.create_subprocess_exec(
        "python", "eval.py",
        env={**__import__("os").environ, **env},
        stdout=asyncio.subprocess.PIPE
    )
    stdout, _ = await proc.communicate()
    score = json.loads(stdout)["score"]
    return {"agent_id": agent_id, "hypothesis": hypothesis, "score": score}

async def parallel_search(hypotheses: list, gpus: list):
    tasks = [
        run_agent(i, gpus[i % len(gpus)], h)
        for i, h in enumerate(hypotheses)
    ]
    results = await asyncio.gather(*tasks)
    best = max(results, key=lambda r: r["score"])
    return best
```

---

## Persistent Memory Across Sessions

```python
# memory.py — frequency-weighted cross-session knowledge retrieval
import json, time
from pathlib import Path

MEMORY_FILE = Path(".autoresearch_memory.json")

def load_memory() -> dict:
    if MEMORY_FILE.exists():
        return json.loads(MEMORY_FILE.read_text())
    return {"lessons": [], "best_score": None, "total_steps": 0}

def save_lesson(lesson: str, score_delta: float):
    mem = load_memory()
    mem["lessons"].append({
        "text": lesson,
        "delta": score_delta,
        "timestamp": time.time(),
        "weight": 1.0
    })
    # Boost weight for high-impact lessons
    if score_delta > 0.01:
        mem["lessons"][-1]["weight"] = 3.0
    MEMORY_FILE.write_text(json.dumps(mem, indent=2))

def get_top_lessons(n: int = 5) -> list[str]:
    mem = load_memory()
    sorted_lessons = sorted(
        mem["lessons"],
        key=lambda l: l["weight"] * l["delta"],
        reverse=True
    )
    return [l["text"] for l in sorted_lessons[:n]]
```

---

## Swarm Coordination ([mutable-state-inc/autoresearch-at-home](https://github.com/mutable-state-inc/autoresearch-at-home))

```python
# swarm.py — share best configs and hypotheses across agents
import json, os, requests

SWARM_API = os.environ.get("SWARM_API_URL", "http://localhost:8080")

def claim_experiment(agent_id: str, hypothesis: str) -> bool:
    """Claim a hypothesis so other agents don't duplicate work."""
    resp = requests.post(f"{SWARM_API}/claim", json={
        "agent_id": agent_id,
        "hypothesis": hypothesis
    })
    return resp.json()["claimed"]

def push_best_config(score: float, config: dict):
    """Broadcast a new best config to the swarm leaderboard."""
    requests.post(f"{SWARM_API}/best", json={
        "score": score,
        "config": config,
        "agent_id": os.environ.get("AGENT_ID", "local")
    })

def pull_best_config() -> dict | None:
    """Fetch current global best config from swarm."""
    resp = requests.get(f"{SWARM_API}/best")
    return resp.json() if resp.ok else None
```

---

## Apple Silicon / MLX Port ([trevin-creator/autoresearch-mlx](https://github.com/trevin-creator/autoresearch-mlx))

```python
# eval_mlx.py — drop-in eval for Apple Silicon (no CUDA required)
import mlx.core as mx
import mlx.nn as nn
import json

def evaluate_mlx(model_path: str, val_data: str) -> float:
    model = nn.load(model_path)          # MLX checkpoint
    tokens = mx.array(load_tokens(val_data))
    logits = model(tokens[:-1])
    loss = nn.losses.cross_entropy(logits, tokens[1:]).mean().item()
    bpb = loss / 0.6931                  # nats → bits
    return bpb

bpb = evaluate_mlx("ckpt.npz", "data/val.bin")
print(json.dumps({"score": -bpb}))       # negate: higher score = lower bpb
```

---

## End-to-End Research Agent ([SakanaAI/AI-Scientist](https://github.com/SakanaAI/AI-Scientist))

```bash
# Clone and install AI-Scientist
git clone https://github.com/SakanaAI/AI-Scientist
cd AI-Scientist
pip install -r requirements.txt

# Set API keys
export OPENAI_API_KEY="..."
export ANTHROPIC_API_KEY="..."

# Run full pipeline: idea → experiments → paper
python launch_scientist.py \
  --model "gpt-4o" \
  --experiment nanoGPT \
  --num-ideas 5
```

---

## Key Environment Variables

```bash
# LLM providers
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...

# Swarm coordination
SWARM_API_URL=http://swarm.internal:8080
AGENT_ID=agent_gpu0

# Hardware
CUDA_VISIBLE_DEVICES=0
PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0   # Apple Silicon

# Loop config
AUTORESEARCH_BUDGET=20
AUTORESEARCH_TARGET_SCORE=0.95
AUTORESEARCH_LOG_FILE=experiments.jsonl
```

---

## Minimal Repo Structure

```
my-autoresearch-project/
├── GOAL.md                  # Objective, metric command, budget
├── CLAUDE.md                # Agent skill/instructions
├── eval.py                  # Returns {"score": float} to stdout
├── train.py                 # Or whatever is being optimized
├── experiments.jsonl        # Append-only experiment log
├── .autoresearch_memory.json  # Cross-session lessons (optional)
└── results/
    └── best_config.json     # Current best configuration
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Agent makes too-large changes | Constrain in GOAL.md: "Edit only one function per step" |
| Eval crashes → always reverts | Wrap eval.py in try/except, return `{"score": -999}` on error |
| No improvement after 10 steps | Lower learning rate, restrict search space, or seed with known-good config |
| GPU OOM during eval | Add `torch.cuda.empty_cache()` before eval; reduce batch size |
| Agent forgets past lessons | Use persistent memory (`.autoresearch_memory.json`) and inject top lessons into context |
| Metric is noisy | Average over 3 runs: `score = mean([measure() for _ in range(3)])` |
| macOS / no CUDA | Use MLX port or set `device = "mps"` in PyTorch |
| Free Colab T4 | Replace Flash Attention 3 with `torch.nn.functional.scaled_dot_product_attention` |

---

## Resources

- [karpathy/autoresearch](https://github.com/karpathy/autoresearch) — original
- [ShengranHu/ADAS](https://github.com/ShengranHu/ADAS) — meta-agent architecture design (ICLR 2025)
- [SakanaAI/AI-Scientist-v2](https://github.com/SakanaAI/AI-Scientist-v2) — template-free scientific discovery
- [HKUDS/AI-Researcher](https://github.com/HKUDS/AI-Researcher) — NeurIPS 2025 end-to-end research automation
- [gepa-ai/gepa](https://github.com/gepa-ai/gepa) — genetic-pareto prompt evolution (ICLR 2026 Oral)
- [snap-stanford/MLAgentBench](https://github.com/snap-stanford/MLAgentBench) — benchmark suite for research agents
```
