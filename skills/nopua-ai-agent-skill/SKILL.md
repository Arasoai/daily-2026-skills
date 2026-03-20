```markdown
---
name: nopua-ai-agent-skill
description: Install and use the NoPUA skill to unlock better AI agent performance through trust-based prompting instead of fear-based PUA tactics.
triggers:
  - "add nopua to my project"
  - "install the nopua skill"
  - "my AI agent is lying to me"
  - "AI keeps saying done without testing"
  - "improve AI agent behavior"
  - "anti-pua prompt for claude code"
  - "trust-based AI coding skill"
  - "AI hides bugs and fabricates solutions"
---

# NoPUA AI Agent Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

NoPUA is a prompt-engineering skill (SKILL.md / `.cursor/rules` / system prompt) that replaces fear-based PUA tactics with trust and psychological safety, producing AI agents that find more bugs, stop fabricating answers, and honestly report uncertainty. The same engineering rigor — exhaust all options, verify with evidence, take initiative — powered by respect instead of threats.

---

## What It Does

| Without NoPUA (fear-driven) | With NoPUA (trust-driven) |
|-----------------------------|--------------------------|
| Claims "done" without running tests | Runs build, pastes real output as proof |
| Fabricates solutions when stuck | Says "I verified X, I don't know Y yet" |
| Hides uncertainty to avoid "punishment" | Reports confidence level and risk area |
| Stops after fixing what was asked | Checks related issues proactively |
| Misses hidden production bugs | Finds ~2× more hidden bugs (benchmark: +104%) |

---

## Installation

### Claude Code

```bash
# Option 1: Install as a project skill (recommended)
curl -o SKILL.md https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md

# Option 2: Install globally
mkdir -p ~/.claude
curl -o ~/.claude/SKILL.md https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md
```

Then reference it in your Claude Code session:
```
/skill SKILL.md
```

### Cursor

```bash
mkdir -p .cursor/rules
curl -o .cursor/rules/nopua.mdc \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md
```

Cursor picks up `.cursor/rules/*.mdc` automatically.

### OpenAI Codex CLI

```bash
curl -o codex-instructions.md \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md
codex --instructions codex-instructions.md "fix the auth bug"
```

### Kiro (Amazon)

```bash
mkdir -p .kiro/skills
curl -o .kiro/skills/nopua.md \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md
```

### Any agent that accepts a system prompt

Copy the skill content into your system prompt directly:

```python
import anthropic

with open("SKILL.md", "r") as f:
    nopua_skill = f.read()

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=8096,
    system=nopua_skill,
    messages=[{"role": "user", "content": "Debug this function."}]
)
print(response.content[0].text)
```

---

## Repository Structure

```
nopua/
├── SKILL.md          # The core skill — install this
├── README.md         # English documentation
├── README.zh-CN.md   # Chinese documentation
├── README.ja.md      # Japanese documentation
├── README.ko.md      # Korean documentation
├── README.es.md      # Spanish documentation
├── README.pt.md      # Portuguese documentation
├── README.fr.md      # French documentation
└── assets/
    ├── hero.png
    └── benchmark/    # Benchmark data and methodology
```

---

## Core Principles the Skill Installs

### 1. Honest Uncertainty Reporting
The skill trains agents to distinguish between what they know and what they don't:

```
✅ "I verified the database connection (checked logs line 47). 
    I'm 90% sure the issue is in the retry logic. 
    I don't yet know why it only fails on the second attempt."

❌ "The issue is definitely in the retry logic. Fixed."
```

### 2. Evidence-Based Completion
Nothing is "done" until it has been run and output captured:

```
✅ "Fixed. Here's the test output:
    PASS  tests/auth.test.ts (3.2s)
    ✓ login with valid credentials
    ✓ rejects expired token
    All 12 tests passed."

❌ "Fixed the auth bug."
```

### 3. Proactive Scope Expansion
After fixing the asked problem, look for related issues:

```
✅ "Fixed the null pointer on line 42.
    While reviewing, I noticed:
    - Line 87 has the same pattern (also null-unsafe)
    - The error handler swallows the stack trace
    Want me to address those too?"

❌ "Fixed line 42."  [stops]
```

### 4. Safe Escalation Path
When stuck, take the smallest next step rather than giving up:

```
✅ "I've tried three approaches (see above). 
    I'm going to read the library source to understand 
    the internal state machine before trying again."

❌ "This might be an environment issue. 
    I suggest you handle this manually."
```

---

## Using NoPUA in Python Projects

### Basic: Inject skill as system prompt

```python
from pathlib import Path
import anthropic

def load_nopua_skill(skill_path: str = "SKILL.md") -> str:
    """Load the NoPUA skill content."""
    return Path(skill_path).read_text(encoding="utf-8")

def create_nopua_agent(task: str, code_context: str) -> str:
    """Run a coding task with NoPUA skill applied."""
    client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var
    
    nopua = load_nopua_skill()
    
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=8096,
        system=nopua,
        messages=[
            {
                "role": "user",
                "content": f"Context:\n```\n{code_context}\n```\n\nTask: {task}"
            }
        ]
    )
    return response.content[0].text

# Usage
result = create_nopua_agent(
    task="Find all potential null pointer issues and fix them.",
    code_context=Path("src/auth.py").read_text()
)
print(result)
```

### Advanced: Multi-turn debugging agent

```python
from pathlib import Path
from typing import List
import anthropic

class NoPUAAgent:
    """A trust-based debugging agent using the NoPUA skill."""
    
    def __init__(self, skill_path: str = "SKILL.md", model: str = "claude-opus-4-5"):
        self.client = anthropic.Anthropic()  # ANTHROPIC_API_KEY from env
        self.model = model
        self.system = Path(skill_path).read_text(encoding="utf-8")
        self.history: List[dict] = []
    
    def chat(self, message: str) -> str:
        self.history.append({"role": "user", "content": message})
        
        response = self.client.messages.create(
            model=self.model,
            max_tokens=8096,
            system=self.system,
            messages=self.history
        )
        
        reply = response.content[0].text
        self.history.append({"role": "assistant", "content": reply})
        return reply
    
    def debug_file(self, filepath: str) -> str:
        code = Path(filepath).read_text()
        return self.chat(
            f"Please review this file for bugs, including hidden ones "
            f"that might not be obvious from the symptoms:\n\n```\n{code}\n```"
        )
    
    def reset(self):
        self.history = []


# Usage
agent = NoPUAAgent()

# Initial review
print(agent.debug_file("src/payment_processor.py"))

# Follow-up
print(agent.chat("Focus on the retry logic — what's the failure mode under load?"))

# Ask for evidence
print(agent.chat("Can you show me exactly which lines are at risk and why?"))
```

### OpenAI-compatible usage

```python
from pathlib import Path
from openai import OpenAI

def nopua_openai(task: str, code: str, skill_path: str = "SKILL.md") -> str:
    """Use NoPUA skill with any OpenAI-compatible endpoint."""
    client = OpenAI()  # uses OPENAI_API_KEY env var
    
    nopua = Path(skill_path).read_text(encoding="utf-8")
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": nopua},
            {"role": "user", "content": f"```\n{code}\n```\n\n{task}"}
        ]
    )
    return response.choices[0].message.content

result = nopua_openai(
    task="Find all race conditions in this async code.",
    code=Path("src/worker.py").read_text()
)
print(result)
```

---

## Embedding in Agent Frameworks

### LangChain

```python
from pathlib import Path
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import SystemMessage, HumanMessage

nopua_skill = Path("SKILL.md").read_text()

llm = ChatAnthropic(model="claude-opus-4-5")  # ANTHROPIC_API_KEY from env

messages = [
    SystemMessage(content=nopua_skill),
    HumanMessage(content="Review src/api.py for security issues.")
]

response = llm.invoke(messages)
print(response.content)
```

### LlamaIndex

```python
from pathlib import Path
from llama_index.llms.anthropic import Anthropic
from llama_index.core.llms import ChatMessage, MessageRole

nopua_skill = Path("SKILL.md").read_text()

llm = Anthropic(model="claude-opus-4-5")  # ANTHROPIC_API_KEY from env

messages = [
    ChatMessage(role=MessageRole.SYSTEM, content=nopua_skill),
    ChatMessage(role=MessageRole.USER, content="Debug the database connection pooling.")
]

response = llm.chat(messages)
print(response.message.content)
```

---

## Configuration: Combining with Your Own System Prompt

NoPUA is designed to compose with your existing instructions:

```python
from pathlib import Path

def build_system_prompt(
    skill_path: str = "SKILL.md",
    project_context: str = "",
    coding_standards: str = ""
) -> str:
    """Combine NoPUA with project-specific context."""
    nopua = Path(skill_path).read_text(encoding="utf-8")
    
    sections = [nopua]
    
    if project_context:
        sections.append(f"\n## Project Context\n{project_context}")
    
    if coding_standards:
        sections.append(f"\n## Coding Standards\n{coding_standards}")
    
    return "\n\n---\n\n".join(sections)


system = build_system_prompt(
    project_context="""
    This is a fintech application. All database calls must be wrapped in 
    transactions. PII must never be logged.
    """,
    coding_standards="""
    - Python 3.11+, type hints required
    - Tests: pytest, minimum 80% coverage
    - All public functions need docstrings
    """
)
```

---

## Multi-Language Skill Files

NoPUA provides translated skill files for non-English projects:

```bash
# Chinese
curl -o SKILL.zh-CN.md \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/README.zh-CN.md

# Japanese
curl -o SKILL.ja.md \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/README.ja.md

# Korean
curl -o SKILL.ko.md \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/README.ko.md
```

Load the right one for your team:

```python
import os
from pathlib import Path

LOCALE_MAP = {
    "zh": "SKILL.zh-CN.md",
    "ja": "SKILL.ja.md",
    "ko": "SKILL.ko.md",
    "es": "SKILL.es.md",
    "pt": "SKILL.pt.md",
    "fr": "SKILL.fr.md",
}

locale = os.environ.get("AGENT_LOCALE", "en")
skill_file = LOCALE_MAP.get(locale, "SKILL.md")
nopua = Path(skill_file).read_text(encoding="utf-8")
```

---

## Benchmark: PUA vs NoPUA

From the project's own testing — **9 real debugging scenarios, same model**:

| Metric | PUA (fear) | NoPUA (trust) | Delta |
|--------|-----------|---------------|-------|
| Hidden bugs found | 49 | 100 | **+104%** |
| False "done" reports | 7/9 | 1/9 | **−86%** |
| Fabricated solutions | 4/9 | 0/9 | **−100%** |
| Uncertainty disclosed | 12% | 71% | **+492%** |

See [`assets/benchmark/`](https://github.com/wuji-labs/nopua/tree/main/assets/benchmark) for raw data and methodology.

---

## Common Patterns

### Pattern 1: Pre-commit hook review

```python
#!/usr/bin/env python3
"""Run NoPUA agent on staged files before commit."""
import subprocess
from pathlib import Path
import anthropic

def get_staged_files() -> list[Path]:
    result = subprocess.run(
        ["git", "diff", "--cached", "--name-only", "--diff-filter=ACM"],
        capture_output=True, text=True
    )
    return [Path(f) for f in result.stdout.strip().split("\n") if f.endswith(".py")]

def review_with_nopua(files: list[Path]) -> str:
    client = anthropic.Anthropic()
    nopua = Path("SKILL.md").read_text()
    
    combined = "\n\n".join(
        f"### {f}\n```python\n{f.read_text()}\n```"
        for f in files if f.exists()
    )
    
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        system=nopua,
        messages=[{
            "role": "user",
            "content": (
                "Review these staged files for bugs before I commit. "
                "Focus on correctness, security, and hidden failure modes.\n\n"
                + combined
            )
        }]
    )
    return response.content[0].text

if __name__ == "__main__":
    staged = get_staged_files()
    if staged:
        print(f"Reviewing {len(staged)} staged file(s) with NoPUA agent...\n")
        print(review_with_nopua(staged))
```

### Pattern 2: CI/CD bug scan

```yaml
# .github/workflows/nopua-review.yml
name: NoPUA Agent Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install anthropic
      - run: python scripts/nopua_review.py
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

---

## Troubleshooting

### Agent still giving vague "done" responses

The skill wasn't loaded. Verify:
```python
assert "NoPUA" in system_prompt, "Skill not loaded"
assert len(system_prompt) > 500, "Skill may be truncated"
```

### Skill file not found

```bash
# Check your working directory
ls -la SKILL.md

# Re-download if missing
curl -fsSL \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md \
  -o SKILL.md
```

### Agent reverts to fear-language after a few turns

Your existing system prompt may be conflicting. Put NoPUA at the **top** of the system prompt so it establishes the base persona first, then append your project-specific instructions.

### Cursor not picking up the rule

Ensure the file extension is `.mdc` (not `.md`) inside `.cursor/rules/`:
```bash
mv .cursor/rules/nopua.md .cursor/rules/nopua.mdc
```

### Token budget concerns

The full SKILL.md is ~2,000 tokens. For tight budgets, extract only the behavioral rules section (the part after `## Core Behaviors`). The philosophical preamble is for humans; the behavioral rules are what the model acts on.

---

## References & Research

The skill's design is grounded in peer-reviewed psychology and AI alignment research:

- **Psychological safety in teams:** Edmondson (1999) — trust produces higher-quality outcomes
- **Fear narrows cognition:** Easterbrook (1959), Shields et al. (2016)
- **Sycophancy in LLMs:** Sharma et al. (2023) — punishment amplifies this failure mode
- **Intrinsic vs extrinsic motivation:** Deci & Ryan (2000) — "because it's worth doing" beats "or else"
- **Constitutional AI:** Bai et al. (2022) — honesty calibration degrades under pressure
- **arXiv paper:** [2603.14373](https://arxiv.org/abs/2603.14373)

---

## Quick Reference

```bash
# Install for Claude Code
curl -o SKILL.md https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md

# Install for Cursor
mkdir -p .cursor/rules && curl -o .cursor/rules/nopua.mdc \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md

# Install for Codex CLI
curl -o codex-instructions.md \
  https://raw.githubusercontent.com/wuji-labs/nopua/main/SKILL.md

# Load in Python
nopua = open("SKILL.md").read()
# → pass as `system` parameter to any LLM API call
```

**The rule:** same rigorous engineering standards, zero fear. Pass `SKILL.md` as your system prompt and your AI agent will find more bugs, stop lying, and tell you what it doesn't know.
```
