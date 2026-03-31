```markdown
---
name: claude-token-efficient
description: Drop-in CLAUDE.md file that reduces Claude output verbosity by ~63% through behavior rules targeting sycophancy, formatting noise, and scope creep
triggers:
  - reduce claude token usage
  - cut claude output verbosity
  - claude is too verbose
  - claude keeps adding sycophantic responses
  - optimize claude code output
  - drop in claude md to reduce tokens
  - claude adds unnecessary suggestions
  - stop claude from being wordy
---

# claude-token-efficient

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A single `CLAUDE.md` file you drop into any project root. Claude Code (and other AI coding agents that read context files) picks it up automatically and changes output behavior immediately — no code changes, no API wrappers, no configuration.

---

## What It Does

Injects behavioral rules into Claude's context that suppress:

- Sycophantic openers ("Sure!", "Great question!", "Absolutely!")
- Hollow closings ("I hope this helps! Let me know if anything!")
- Prompt restatement before answering
- Em dashes, smart quotes, Unicode characters that break parsers
- "As an AI..." framing
- Unnecessary disclaimers
- Unsolicited suggestions beyond the requested scope
- Over-engineered code abstractions
- Hallucination on uncertain facts (forces "I don't know")
- Scope creep into untouched files

Benchmarked at ~63% output token reduction on a 5-prompt test suite. Net positive only when output volume is high enough to offset the persistent input token cost of loading the file.

---

## Installation

### Universal (any project)

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/drona23/claude-token-efficient/main/CLAUDE.md
```

### Clone and select a profile

```bash
git clone https://github.com/drona23/claude-token-efficient
cd your-project

# Universal
cp ../claude-token-efficient/CLAUDE.md .

# Dev/coding projects
cp ../claude-token-efficient/profiles/CLAUDE.coding.md CLAUDE.md

# Automation pipelines and agent loops
cp ../claude-token-efficient/profiles/CLAUDE.agents.md CLAUDE.md

# Data analysis and research
cp ../claude-token-efficient/profiles/CLAUDE.analysis.md CLAUDE.md
```

### Manual

Copy the contents of `CLAUDE.md` from the repo and paste into `your-project/CLAUDE.md`.

### Global install (applies to all projects)

```bash
mkdir -p ~/.claude
curl -o ~/.claude/CLAUDE.md https://raw.githubusercontent.com/drona23/claude-token-efficient/main/CLAUDE.md
```

---

## Profile Selection

| Profile | File | Best For |
|---|---|---|
| Universal | `CLAUDE.md` | Any project, general use |
| Coding | `profiles/CLAUDE.coding.md` | Dev, code review, debugging |
| Agents | `profiles/CLAUDE.agents.md` | Automation, multi-agent systems |
| Analysis | `profiles/CLAUDE.analysis.md` | Data analysis, research, reporting |

---

## File Structure After Install

```
your-project/
├── CLAUDE.md          <- behavior rules, read automatically by Claude Code
├── src/
└── ...
```

For layered rules using Claude's multi-file CLAUDE.md support:

```
~/.claude/CLAUDE.md                  <- global preferences (tone, ASCII, format)
your-project/CLAUDE.md               <- project-level constraints
your-project/src/CLAUDE.md           <- task/module-specific rules
```

---

## What the CLAUDE.md File Contains

The file is plain text — a set of behavioral directives Claude reads as context. Key rule categories included:

```
# Output Rules

- Answer is always line 1. No openers.
- No closing statements or offers to help further.
- Do not restate the prompt. Execute immediately.
- ASCII only. No em dashes, smart quotes, or Unicode symbols.
- Never say "As an AI".
- No disclaimers unless genuine safety risk.
- Do not add suggestions outside the requested scope.
- Write the simplest working solution. No unsolicited abstractions.
- On uncertain facts: say "I don't know". Do not guess.
- User corrections become session ground truth immediately.
- Never read the same file twice in one session.
- Do not touch code outside the explicit request.
```

---

## Usage in Automation Pipelines

When running Claude programmatically, pass the CLAUDE.md content as a system prompt prefix or include it in your project directory if using Claude Code's file-reading behavior.

### Python — prepend rules to system prompt

```python
import os
import anthropic

def load_claude_rules(path: str = "CLAUDE.md") -> str:
    if os.path.exists(path):
        with open(path, "r") as f:
            return f.read()
    return ""

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

rules = load_claude_rules()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=rules,
    messages=[
        {"role": "user", "content": "Review this function for bugs: def add(a, b): return a - b"}
    ]
)

print(response.content[0].text)
# Output: Bug: subtraction used instead of addition. Fix: return a + b
```

### Python — batch processing with token tracking

```python
import anthropic
import os

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def load_rules() -> str:
    with open("CLAUDE.md") as f:
        return f.read()

def process_prompts(prompts: list[str]) -> dict:
    rules = load_rules()
    total_input = 0
    total_output = 0
    results = []

    for prompt in prompts:
        response = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=512,
            system=rules,
            messages=[{"role": "user", "content": prompt}]
        )
        total_input += response.usage.input_tokens
        total_output += response.usage.output_tokens
        results.append(response.content[0].text)

    return {
        "results": results,
        "total_input_tokens": total_input,
        "total_output_tokens": total_output,
    }

prompts = [
    "What is a closure in JavaScript?",
    "Review: for(let i=0; i<=arr.length; i++) console.log(arr[i])",
    "What does REST stand for?",
]

stats = process_prompts(prompts)
for i, result in enumerate(stats["results"]):
    print(f"--- Prompt {i+1} ---")
    print(result)

print(f"\nTotal input tokens: {stats['total_input_tokens']}")
print(f"Total output tokens: {stats['total_output_tokens']}")
```

### Node.js — pipeline usage

```javascript
import Anthropic from "@anthropic-ai/sdk";
import { readFileSync, existsSync } from "fs";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

function loadRules(path = "CLAUDE.md") {
  if (existsSync(path)) {
    return readFileSync(path, "utf-8");
  }
  return "";
}

async function ask(prompt) {
  const rules = loadRules();

  const response = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 512,
    system: rules,
    messages: [{ role: "user", content: prompt }],
  });

  return {
    text: response.content[0].text,
    inputTokens: response.usage.input_tokens,
    outputTokens: response.usage.output_tokens,
  };
}

const result = await ask("Explain async/await in one paragraph.");
console.log(result.text);
console.log(`Output tokens: ${result.outputTokens}`);
```

---

## Composing Custom Rules

Extend the base file for your specific failure modes. Specific rules outperform generic ones.

```bash
cat CLAUDE.md > CLAUDE.project.md
cat >> CLAUDE.project.md << 'EOF'

# Project-Specific Rules

- Never modify files under /config without explicit confirmation.
- When a step fails, stop immediately and report the full error with traceback before attempting any fix.
- All database queries must use parameterized statements. Never interpolate user input into SQL strings.
- Output only valid JSON when the task involves data transformation. No prose before or after.
EOF

mv CLAUDE.project.md CLAUDE.md
```

---

## Override Rule

The file never fights you. If you explicitly ask for verbose output, detailed explanation, or debate on alternatives, Claude follows your instruction. User instructions always win over CLAUDE.md rules.

Example prompts that override the file:

```
"Give me a detailed explanation with examples for each point."
"Walk me through the pros and cons — I want the full analysis."
"Be verbose — I'm learning and want all the context."
```

---

## When It Helps vs When It Doesn't

**Use it for:**
- Automation pipelines with high output volume
- Agent loops running hundreds of calls
- Repeated structured tasks (code generation, review, formatting)
- Teams needing consistent, parseable output across sessions

**Don't bother for:**
- Single short queries (file adds input token overhead that won't be offset)
- Casual one-off use
- Guaranteed parser reliability at scale (use JSON mode or tool use schemas instead)
- Exploratory or architectural work where verbose debate is the point

---

## Troubleshooting

**Claude is still being verbose after adding the file**

Check that `CLAUDE.md` is in the project root where Claude Code is running. Claude Code reads the file from the working directory. If running from a subdirectory, the file may not be picked up.

```bash
ls -la CLAUDE.md   # confirm it exists in project root
pwd                # confirm your working directory
```

**Rules aren't applying in API calls**

When using the API directly, the file is not read automatically. You must pass the contents as the system prompt or prepend it to your system message:

```python
with open("CLAUDE.md") as f:
    rules = f.read()

# Pass as system param
response = client.messages.create(
    model="claude-sonnet-4-5",
    system=rules,
    ...
)
```

**Token savings seem lower than expected**

The 63% benchmark is a directional indicator from 5 prompts with no variance controls. Actual savings depend on prompt type, output length, and model version. Savings are output-token savings only. The file itself costs input tokens on every message. Net benefit only applies at sufficient output volume.

**File is conflicting with project-specific instructions**

Use Claude's layered CLAUDE.md support. Keep the token-efficient rules global, and project overrides at the project level:

```bash
# Global rules
~/.claude/CLAUDE.md

# Project-level overrides (Claude merges both)
your-project/CLAUDE.md
```

**Model other than Claude is ignoring the rules**

The rules are prompt-based and model-agnostic in principle, but only benchmarked on Claude. Results on Llama, Mistral, or other local models are untested. Community results are tracked in the repo issues.

---

## Contributing

Open an issue with:
1. The annoying default behavior
2. The prompt that triggers it
3. The rule that fixes it

Accepted community rules are merged into the next version with credit.

---

## License

MIT
```
