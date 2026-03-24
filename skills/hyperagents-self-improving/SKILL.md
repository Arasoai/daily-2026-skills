```markdown
---
name: hyperagents-self-improving
description: Self-referential self-improving agents from Meta Research that optimize for any computable task using meta-agents and task-agents in a recursive loop
triggers:
  - set up hyperagents
  - run self-improving agent loop
  - configure meta agent for domain
  - use hyperagents to optimize a task
  - implement self-referential agent
  - run generate loop with hyperagents
  - hyperagents experiment setup
  - facebookresearch hyperagents
---

# HyperAgents Self-Improving Agents

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

HyperAgents is a Meta Research framework for self-referential, self-improving agents that recursively optimize themselves for any computable task. A **meta-agent** proposes code changes (diffs) to improve a **task-agent**, which is then evaluated on a target domain. The loop continues, progressively improving agent performance.

## Architecture Overview

```
generate_loop.py
    └── meta_agent.py       ← proposes improvements (diffs) to task agent code
    └── task_agent.py       ← executes tasks in a target domain
    └── agent/              ← foundation model wrappers (OpenAI, Anthropic, Gemini)
    └── domains/            ← domain-specific evaluation code
    └── run_meta_agent.py   ← helper to run meta agent and get diffs
```

The meta-agent reads the current task-agent source, generates improved versions, applies diffs, and evaluates the new agent. This is repeated in a loop.

---

## Installation

### 1. System Dependencies (Fedora/RHEL)
```bash
sudo dnf install -y python3.12-devel
sudo dnf install -y graphviz graphviz-devel cmake ninja-build \
    bzip2-devel zlib-devel ncurses-devel libffi-devel
```

### 2. Python Environment
```bash
python3.12 -m venv venv_nat
source venv_nat/bin/activate
pip install -r requirements.txt
pip install -r requirements_dev.txt
```

### 3. API Keys
Create a `.env` file in the project root:
```bash
OPENAI_API_KEY=your_openai_key_here
ANTHROPIC_API_KEY=your_anthropic_key_here
GEMINI_API_KEY=your_gemini_key_here
```

### 4. Initial Agent Setup
```bash
bash ./setup_initial.sh
```

### 5. (Optional) Docker
```bash
docker build --network=host -t hyperagents .
```

> ⚠️ **Safety Warning**: HyperAgents executes untrusted, model-generated code. Run in an isolated environment (Docker, VM, or sandboxed container). Never run on a production system.

---

## Key Commands

### Run the Main Loop
```bash
# Basic run on a domain
python generate_loop.py --domains <domain>

# Examples:
python generate_loop.py --domains coding
python generate_loop.py --domains math
python generate_loop.py --domains reasoning
```

### Run the Meta Agent Standalone
```bash
python run_meta_agent.py
```

### Extract Experiment Logs
```bash
# Combine multi-part ZIP archive
zip -s 0 outputs_os_parts.zip --out unsplit_logs.zip
unzip unsplit_outputs.zip
```

Outputs are saved in `outputs/` by default.

---

## Core Files and Usage

### `generate_loop.py` — Entry Point

```python
# Typical invocation pattern (from source)
import argparse
from meta_agent import MetaAgent
from task_agent import TaskAgent

parser = argparse.ArgumentParser()
parser.add_argument("--domains", type=str, required=True)
parser.add_argument("--output_dir", type=str, default="outputs/")
parser.add_argument("--num_iterations", type=int, default=10)
parser.add_argument("--model", type=str, default="gpt-4o")
args = parser.parse_args()
```

### `meta_agent.py` — Meta Agent

The meta-agent reads task-agent code and proposes improvements:

```python
from meta_agent import MetaAgent

meta = MetaAgent(
    model="claude-3-5-sonnet-20241022",  # or "gpt-4o", "gemini-pro"
    task_description="Solve competitive programming problems",
    current_agent_code=open("task_agent.py").read()
)

# Get proposed diff/improvement
diff = meta.propose_improvement()
print(diff)
```

### `task_agent.py` — Task Agent

The task-agent executes tasks in a domain:

```python
from task_agent import TaskAgent

agent = TaskAgent(
    model="gpt-4o",
    domain="coding"
)

# Run agent on a task
result = agent.run(task_input="Solve: given an array, find the maximum subarray sum")
print(result)
```

### `agent/` — Foundation Model Wrappers

```python
# Using OpenAI wrapper
from agent.openai_agent import OpenAIAgent

agent = OpenAIAgent(model="gpt-4o")
response = agent.generate(
    system_prompt="You are a helpful coding assistant.",
    user_prompt="Write a binary search implementation in Python."
)

# Using Anthropic wrapper
from agent.anthropic_agent import AnthropicAgent

agent = AnthropicAgent(model="claude-3-5-sonnet-20241022")
response = agent.generate(
    system_prompt="You are a helpful assistant.",
    user_prompt="Explain self-referential improvement."
)

# Using Gemini wrapper
from agent.gemini_agent import GeminiAgent

agent = GeminiAgent(model="gemini-pro")
response = agent.generate(prompt="Optimize this Python function for speed.")
```

---

## Domain Configuration

Domains live in `domains/`. Each domain defines how tasks are sampled and how agents are evaluated:

```python
# Example domain structure (domains/coding/)
# domains/coding/__init__.py
# domains/coding/tasks.py      ← task definitions
# domains/coding/evaluator.py  ← scoring/evaluation logic

from domains.coding.evaluator import CodingEvaluator

evaluator = CodingEvaluator()
score = evaluator.evaluate(
    agent_output="def max_subarray(arr): ...",
    ground_truth_tests=[...]
)
```

To add a custom domain:
1. Create `domains/my_domain/` directory
2. Implement `tasks.py` with task generation
3. Implement `evaluator.py` with scoring logic
4. Register the domain in `generate_loop.py`

---

## Common Patterns

### Pattern 1: Full Self-Improvement Loop

```python
import os
from dotenv import load_dotenv
from meta_agent import MetaAgent
from task_agent import TaskAgent

load_dotenv()

DOMAIN = "coding"
NUM_ITERATIONS = 5
META_MODEL = "claude-3-5-sonnet-20241022"
TASK_MODEL = "gpt-4o"

# Load initial task agent source
with open("task_agent.py", "r") as f:
    agent_code = f.read()

meta = MetaAgent(model=META_MODEL)
scores = []

for i in range(NUM_ITERATIONS):
    print(f"\n=== Iteration {i+1}/{NUM_ITERATIONS} ===")
    
    # Meta-agent proposes improvement
    improved_code = meta.propose_improvement(
        current_code=agent_code,
        domain=DOMAIN,
        iteration=i
    )
    
    # Evaluate improved agent
    task_agent = TaskAgent(model=TASK_MODEL, code=improved_code)
    score = task_agent.evaluate_on_domain(DOMAIN, num_tasks=10)
    scores.append(score)
    print(f"Score: {score:.3f}")
    
    # Accept improvement if score is better
    if score > max(scores[:-1], default=0):
        agent_code = improved_code
        print("✓ Improvement accepted")
    else:
        print("✗ Improvement rejected, keeping previous version")
```

### Pattern 2: Running With Docker (Recommended for Safety)

```bash
# Build image
docker build --network=host -t hyperagents .

# Run with env file
docker run --env-file .env \
    -v $(pwd)/outputs:/app/outputs \
    hyperagents python generate_loop.py --domains coding

# Interactive shell inside container
docker run -it --env-file .env hyperagents /bin/bash
```

### Pattern 3: Analyzing Results

```python
import json
import os

output_dir = "outputs/"

# Load all iteration results
results = []
for fname in sorted(os.listdir(output_dir)):
    if fname.endswith(".json"):
        with open(os.path.join(output_dir, fname)) as f:
            results.append(json.load(f))

# Plot improvement curve
import matplotlib.pyplot as plt

iterations = [r["iteration"] for r in results]
scores = [r["score"] for r in results]

plt.plot(iterations, scores, marker='o')
plt.xlabel("Iteration")
plt.ylabel("Score")
plt.title("HyperAgents Self-Improvement Curve")
plt.savefig("improvement_curve.png")
plt.show()
```

### Pattern 4: Selecting Baseline vs. HyperAgents

```bash
# Run with a specific baseline (see generate_loop.py --help for options)
python generate_loop.py --domains coding --baseline random
python generate_loop.py --domains coding --baseline zero_shot
python generate_loop.py --domains coding  # default: HyperAgents loop
```

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENAI_API_KEY` | Conditional | Required if using GPT models |
| `ANTHROPIC_API_KEY` | Conditional | Required if using Claude models |
| `GEMINI_API_KEY` | Conditional | Required if using Gemini models |

Load via `.env` file or export directly:
```bash
export OPENAI_API_KEY=$(cat ~/.secrets/openai_key)
```

---

## Troubleshooting

### `graphviz` import error
```bash
sudo dnf install -y graphviz graphviz-devel
pip install --force-reinstall pygraphviz
```

### `python3.12-devel` not found (Ubuntu/Debian)
```bash
sudo apt-get install python3.12-dev
sudo apt-get install libgraphviz-dev cmake ninja-build
```

### API rate limit errors
- Add retry logic or reduce `--num_iterations`
- Switch to a different model provider in your config

### Generated code causes crashes
- This is expected behavior — the framework catches exceptions and rejects bad improvements
- Run inside Docker for full isolation
- Check `outputs/` for logs of what code was generated

### `.env` not being loaded
```bash
pip install python-dotenv
# Ensure load_dotenv() is called at the top of your entry script
```

### Extracting multi-part experiment logs
```bash
# All parts (.z01, .z02, .zip) must be in the same directory
zip -s 0 outputs_os_parts.zip --out unsplit_logs.zip
unzip unsplit_outputs.zip -d logs/
```

---

## Citation

```bibtex
@misc{zhang2026hyperagents,
      title={Hyperagents}, 
      author={Jenny Zhang and Bingchen Zhao and Wannan Yang and Jakob Foerster 
              and Jeff Clune and Minqi Jiang and Sam Devlin and Tatiana Shavrina},
      year={2026},
      eprint={2603.19461},
      archivePrefix={arXiv},
      primaryClass={cs.AI},
      url={https://arxiv.org/abs/2603.19461}, 
}
```

**Links**: [arXiv Paper](https://arxiv.org/abs/2603.19461) | [Meta AI Blog](https://ai.meta.com/research/publications/hyperagents/) | [License: CC BY-NC-SA 4.0](LICENSE.md)
```
