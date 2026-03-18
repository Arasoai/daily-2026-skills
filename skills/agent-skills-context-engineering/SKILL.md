```markdown
---
name: agent-skills-context-engineering
description: Comprehensive collection of Agent Skills for context engineering, multi-agent architectures, memory systems, and production agent systems using Claude Code, Cursor, or any agent platform.
triggers:
  - "context engineering for agents"
  - "build multi-agent system"
  - "manage agent context window"
  - "install agent skills claude code"
  - "optimize token usage in agents"
  - "design memory system for agent"
  - "LLM context degradation lost in middle"
  - "evaluate agent performance with LLM judge"
---

# Agent Skills for Context Engineering

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A comprehensive, open collection of Agent Skills focused on context engineering principles for building production-grade AI agent systems. Teaches the art and science of curating context to maximize agent effectiveness across Claude Code, Cursor, Codex, and any agent platform.

## What This Project Does

Context engineering is the discipline of managing a language model's context window holistically — system prompts, tool definitions, retrieved documents, message history, and tool outputs. As context grows, models exhibit predictable degradation (lost-in-the-middle, U-shaped attention, attention scarcity). This repository provides structured skills to address all of these problems.

Skills are organized into four categories:
- **Foundational** — context fundamentals, degradation patterns, compression
- **Architectural** — multi-agent patterns, memory systems, tool design, filesystem context, hosted agents
- **Operational** — context optimization, evaluation, advanced LLM-as-judge evaluation
- **Development Methodology** — project development, BDI cognitive architectures

---

## Installation

### Claude Code (Plugin Marketplace)

```bash
# Register the marketplace
/plugin marketplace add muratcankoylan/Agent-Skills-for-Context-Engineering

# Install individual plugin bundles
/plugin install context-engineering-fundamentals@context-engineering-marketplace
/plugin install agent-architecture@context-engineering-marketplace
/plugin install agent-evaluation@context-engineering-marketplace
/plugin install agent-development@context-engineering-marketplace
/plugin install cognitive-architecture@context-engineering-marketplace
```

### Cursor

Listed on [Cursor Plugin Directory](https://cursor.directory/plugins/context-engineering). The `.plugin/plugin.json` manifest follows the Open Plugins standard — works with Codex, GitHub Copilot, and any conformant agent tool.

### Manual / Custom Agent

Clone the repo and reference skill files directly:

```bash
git clone https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering.git
```

Load a skill's `SKILL.md` (or equivalent) into your agent's system prompt or context as needed.

---

## Plugin Bundles → Skills Mapping

| Plugin | Skills Included |
|--------|-----------------|
| `context-engineering-fundamentals` | context-fundamentals, context-degradation, context-compression, context-optimization |
| `agent-architecture` | multi-agent-patterns, memory-systems, tool-design, filesystem-context, hosted-agents |
| `agent-evaluation` | evaluation, advanced-evaluation |
| `agent-development` | project-development |
| `cognitive-architecture` | bdi-mental-states |

---

## Skill Trigger Reference

Use these phrases to activate specific skills automatically:

| Skill | Activation Phrases |
|-------|--------------------|
| `context-fundamentals` | "understand context", "explain context windows", "design agent architecture" |
| `context-degradation` | "diagnose context problems", "fix lost-in-middle", "debug agent failures" |
| `context-compression` | "compress context", "summarize conversation", "reduce token usage" |
| `context-optimization` | "optimize context", "reduce token costs", "implement KV-cache" |
| `multi-agent-patterns` | "design multi-agent system", "implement supervisor pattern" |
| `memory-systems` | "implement agent memory", "build knowledge graph", "track entities" |
| `tool-design` | "design agent tools", "reduce tool complexity", "implement MCP tools" |
| `filesystem-context` | "offload context to files", "dynamic context discovery", "agent scratch pad" |
| `hosted-agents` | "build background agent", "sandboxed execution", "multiplayer agent", "Modal sandboxes" |
| `evaluation` | "evaluate agent performance", "build test framework", "measure quality" |
| `advanced-evaluation` | "implement LLM-as-judge", "compare model outputs", "mitigate bias" |
| `project-development` | "start LLM project", "design batch pipeline", "evaluate task-model fit" |
| `bdi-mental-states` | "model agent mental states", "implement BDI architecture", "transform RDF to beliefs" |

---

## Repository Structure

```
Agent-Skills-for-Context-Engineering/
├── .plugin/
│   └── plugin.json              # Open Plugins manifest
├── skills/
│   ├── context-fundamentals/
│   ├── context-degradation/
│   ├── context-compression/
│   ├── context-optimization/
│   ├── multi-agent-patterns/
│   ├── memory-systems/
│   ├── tool-design/
│   ├── filesystem-context/
│   ├── hosted-agents/
│   ├── evaluation/
│   ├── advanced-evaluation/
│   ├── project-development/
│   └── bdi-mental-states/
└── examples/
    ├── digital-brain-skill/
    ├── x-to-book-system/
    ├── llm-as-judge-skills/
    └── book-sft-pipeline/
```

---

## Core Concepts with Code Examples

### 1. Context Compression (Python)

```python
# Pattern: Compress long conversation history before appending new turns
from anthropic import Anthropic

client = Anthropic()

def compress_history(messages: list[dict], max_tokens: int = 2000) -> list[dict]:
    """
    Summarize older messages when approaching context limits.
    Preserves the last N turns verbatim; summarizes everything older.
    """
    if len(messages) <= 4:
        return messages

    # Keep the last 4 messages verbatim
    recent = messages[-4:]
    older = messages[:-4]

    # Build a summary of older context
    summary_prompt = (
        "Summarize the following conversation history concisely, "
        "preserving all key decisions, facts, and intent:\n\n"
        + "\n".join(f"{m['role']}: {m['content']}" for m in older)
    )

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=max_tokens,
        messages=[{"role": "user", "content": summary_prompt}],
    )
    summary_text = response.content[0].text

    compressed = [
        {"role": "user", "content": f"[Conversation summary]: {summary_text}"},
        {"role": "assistant", "content": "Understood. Continuing from the summary."},
    ]
    return compressed + recent
```

---

### 2. Multi-Agent Orchestrator Pattern (Python)

```python
# Pattern: Orchestrator dispatches subtasks to specialist agents
import anthropic
from dataclasses import dataclass

client = anthropic.Anthropic()

@dataclass
class AgentTask:
    task_id: str
    description: str
    specialist: str  # "researcher" | "writer" | "critic"

def specialist_agent(task: AgentTask, context: str) -> str:
    """A specialist agent that receives only relevant context."""
    system_prompts = {
        "researcher": "You are a research specialist. Find facts, cite sources, be precise.",
        "writer": "You are a writing specialist. Produce clear, structured prose.",
        "critic": "You are a critic. Identify flaws, gaps, and improvements.",
    }
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=system_prompts[task.specialist],
        messages=[
            {"role": "user", "content": f"Context:\n{context}\n\nTask: {task.description}"}
        ],
    )
    return response.content[0].text

def orchestrator(goal: str, tasks: list[AgentTask]) -> dict[str, str]:
    """
    Orchestrator: assigns tasks, collects outputs, never passes full shared
    context to every agent — only what each specialist needs.
    """
    results = {}
    shared_context = f"Overall goal: {goal}"

    for task in tasks:
        # Progressive context: each agent only sees prior results relevant to it
        relevant_context = shared_context
        if task.specialist == "writer" and "researcher" in results:
            relevant_context += f"\n\nResearch findings:\n{results['researcher']}"
        elif task.specialist == "critic" and "writer" in results:
            relevant_context += f"\n\nDraft:\n{results['writer']}"

        results[task.specialist] = specialist_agent(task, relevant_context)
        shared_context += f"\n\n[{task.specialist} output]: {results[task.specialist][:200]}..."

    return results

# Usage
goal = "Write a technical blog post about transformer attention mechanisms"
tasks = [
    AgentTask("t1", "Research key papers on transformer attention", "researcher"),
    AgentTask("t2", "Write a 500-word blog post based on the research", "writer"),
    AgentTask("t3", "Review the draft for technical accuracy", "critic"),
]
outputs = orchestrator(goal, tasks)
```

---

### 3. Memory System: Append-Only JSONL (Python)

```python
# Pattern: Append-only memory with schema-first line for agent-friendly parsing
import json
from pathlib import Path
from datetime import datetime, timezone

MEMORY_FILE = Path("agent_memory.jsonl")

# Schema definition (first line of file — written once)
SCHEMA = {
    "_schema": True,
    "fields": ["timestamp", "type", "content", "tags"],
    "version": "1.0"
}

def init_memory():
    if not MEMORY_FILE.exists():
        with open(MEMORY_FILE, "w") as f:
            f.write(json.dumps(SCHEMA) + "\n")

def remember(entry_type: str, content: str, tags: list[str] = None):
    """Append a memory entry."""
    init_memory()
    entry = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "type": entry_type,  # "fact" | "decision" | "entity" | "preference"
        "content": content,
        "tags": tags or [],
    }
    with open(MEMORY_FILE, "a") as f:
        f.write(json.dumps(entry) + "\n")

def recall(entry_type: str = None, tags: list[str] = None, limit: int = 50) -> list[dict]:
    """Read memory, optionally filtered. Skips schema line."""
    if not MEMORY_FILE.exists():
        return []
    entries = []
    with open(MEMORY_FILE) as f:
        for line in f:
            entry = json.loads(line)
            if entry.get("_schema"):
                continue
            if entry_type and entry["type"] != entry_type:
                continue
            if tags and not any(t in entry["tags"] for t in tags):
                continue
            entries.append(entry)
    return entries[-limit:]

# Usage
remember("decision", "Use JSONL for memory — append-only, agent-parseable", ["architecture"])
remember("fact", "Context windows degrade with lost-in-middle pattern", ["context", "research"])

decisions = recall(entry_type="decision")
context_facts = recall(tags=["context"])
```

---

### 4. Tool Design: Minimal Surface Area (Python)

```python
# Pattern: Tools should have narrow, predictable outputs to avoid context bloat
import anthropic
import json

client = anthropic.Anthropic()

# BAD: Tool returns full raw data (bloats context)
def search_bad(query: str) -> str:
    results = _do_search(query)
    return json.dumps(results)  # Could be thousands of tokens

# GOOD: Tool returns minimal, structured summary
def search_good(query: str, top_k: int = 3) -> str:
    results = _do_search(query)
    summaries = [
        {"title": r["title"], "snippet": r["snippet"][:150], "url": r["url"]}
        for r in results[:top_k]
    ]
    return json.dumps(summaries)  # Controlled, bounded output

tools = [
    {
        "name": "web_search",
        "description": (
            "Search the web. Returns top 3 results with title, 150-char snippet, and URL. "
            "Use for factual lookups only. Do not use for code or math questions."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "The search query"},
                "top_k": {"type": "integer", "default": 3, "description": "Number of results (max 5)"},
            },
            "required": ["query"],
        },
    }
]

def run_agent_with_tools(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            return response.content[0].text

        if response.stop_reason == "tool_use":
            tool_use = next(b for b in response.content if b.type == "tool_use")
            tool_result = search_good(**tool_use.input)

            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{"type": "tool_result", "tool_use_id": tool_use.id, "content": tool_result}],
            })

def _do_search(query: str) -> list[dict]:
    # Placeholder — replace with real search integration
    return [{"title": f"Result for {query}", "snippet": "...", "url": "https://example.com"}]
```

---

### 5. LLM-as-Judge Evaluation (Python)

```python
# Pattern: Direct scoring with rubric to evaluate agent outputs
import anthropic
import json

client = anthropic.Anthropic()

JUDGE_SYSTEM = """You are an impartial evaluator. Score responses on the provided rubric.
Return ONLY valid JSON: {"score": <0-10>, "reasoning": "<one sentence>", "pass": <true|false>}"""

def evaluate_response(
    question: str,
    response: str,
    rubric: str,
    pass_threshold: float = 7.0,
) -> dict:
    """
    Direct scoring evaluation.
    Rubric example: "Score on factual accuracy (0-10). Deduct 2pts per factual error."
    """
    prompt = f"""Question: {question}

Response to evaluate:
{response}

Rubric: {rubric}

Evaluate and return JSON."""

    result = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=256,
        system=JUDGE_SYSTEM,
        messages=[{"role": "user", "content": prompt}],
    )
    evaluation = json.loads(result.content[0].text)
    evaluation["pass"] = evaluation["score"] >= pass_threshold
    return evaluation

def pairwise_compare(question: str, response_a: str, response_b: str) -> dict:
    """
    Pairwise comparison with position bias mitigation (run both orderings).
    """
    def _compare(first: str, second: str) -> str:
        prompt = f"""Question: {question}

Response A: {first}

Response B: {second}

Which response is better? Return JSON: {{"winner": "A" or "B" or "tie", "reason": "..."}}"""
        result = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=128,
            messages=[{"role": "user", "content": prompt}],
        )
        return json.loads(result.content[0].text)

    # Run both orderings to cancel position bias
    forward = _compare(response_a, response_b)
    backward = _compare(response_b, response_a)

    # Normalize backward result (A/B labels are swapped)
    backward_normalized = "A" if backward["winner"] == "B" else (
        "B" if backward["winner"] == "A" else "tie"
    )

    if forward["winner"] == backward_normalized:
        return {"winner": forward["winner"], "confidence": "high", "reason": forward["reason"]}
    return {"winner": "tie", "confidence": "low", "reason": "Position bias detected — inconclusive"}

# Usage
score = evaluate_response(
    question="What causes the lost-in-the-middle problem?",
    response="Models pay less attention to content in the middle of long contexts due to attention mechanics.",
    rubric="Score on factual accuracy and completeness (0-10). 10=fully correct and complete.",
)
print(score)  # {"score": 8, "reasoning": "Correct but omits U-shaped curve detail.", "pass": true}
```

---

### 6. Filesystem Context Offloading (Python)

```python
# Pattern: Offload large tool outputs to files; pass only file references in context
import tempfile
import json
from pathlib import Path

SCRATCH_DIR = Path(tempfile.gettempdir()) / "agent_scratch"
SCRATCH_DIR.mkdir(exist_ok=True)

def offload_to_file(data: dict | list | str, label: str) -> str:
    """
    Write large data to a scratch file. Return a compact context reference.
    Prevents tool outputs from filling the context window.
    """
    file_path = SCRATCH_DIR / f"{label}.json"
    content = json.dumps(data, indent=2) if not isinstance(data, str) else data
    file_path.write_text(content)
    token_estimate = len(content.split()) * 1.3
    return f"[FILE:{file_path}] ({int(token_estimate)} est. tokens — load only if needed)"

def load_from_file(file_ref: str) -> str:
    """Parse a file reference and load content."""
    path_str = file_ref.split("[FILE:")[1].split("]")[0]
    return Path(path_str).read_text()

def dynamic_context_discovery(agent_workspace: Path) -> str:
    """
    Scan workspace for agent-relevant files. Return a compact manifest.
    Agents load this manifest instead of all file contents upfront.
    """
    manifest = []
    for f in agent_workspace.rglob("*"):
        if f.is_file() and f.suffix in {".md", ".json", ".jsonl", ".py", ".txt"}:
            size = f.stat().st_size
            manifest.append({
                "path": str(f.relative_to(agent_workspace)),
                "size_bytes": size,
                "load_hint": "on-demand" if size > 5000 else "safe-to-inline",
            })
    return json.dumps({"workspace_manifest": manifest}, indent=2)

# Usage
large_api_result = {"items": list(range(1000))}  # Would be ~3000 tokens inline
ref = offload_to_file(large_api_result, "api_result_page1")
# Pass `ref` to agent context — agent requests load only when needed
print(ref)  # [FILE:/tmp/agent_scratch/api_result_page1.json] (1300 est. tokens — load only if needed)
```

---

## Common Patterns

### Progressive Disclosure (3-Level Loading)

```
Level 1 (always loaded): Skill name + one-line description (~50 tokens)
Level 2 (on activation): Module overview and key APIs (~500 tokens)
Level 3 (on demand): Full implementation details, examples (~5000 tokens)
```

Implement this in your agent by structuring SKILL.md files with collapsible sections or by using separate files per level.

### KV-Cache Optimization

```python
# Place static, rarely-changing content at the TOP of system prompt
# to maximize KV-cache hits across turns.
system_prompt = """
[STATIC - cache-friendly]
You are an expert context engineering assistant.
Rules: ...
Tool definitions: ...

[DYNAMIC - append below]
Current task context: {task_context}
Recent observations: {observations}
"""
```

### Attention Budget Allocation

```
Context Window Budget (example: 200k tokens)
├── System prompt + tools:   ~5%   (10k)  — keep minimal
├── Retrieved documents:    ~30%   (60k)  — ranked by relevance
├── Conversation history:   ~20%   (40k)  — compressed older turns
├── Tool outputs:           ~15%   (30k)  — offload large outputs to files
└── Working space (output): ~30%   (60k)  — reserve for generation
```

---

## Troubleshooting

### Agent "forgets" earlier instructions

**Cause**: Lost-in-the-middle — instructions buried in context center.
**Fix**: Move critical instructions to the beginning AND end of system prompt. Compress middle history.

```python
system = f"""
CRITICAL RULES (always follow):
{critical_rules}

[Background context — may be deprioritized by attention]
{background_info}

REMINDER — CRITICAL RULES STILL APPLY:
{critical_rules_short_repeat}
"""
```

### Context growing unbounded in long sessions

**Cause**: Appending all turns without compression.
**Fix**: Apply rolling compression after every N turns.

```python
MAX_TURNS_BEFORE_COMPRESS = 10

def maybe_compress(messages: list[dict]) -> list[dict]:
    if len(messages) > MAX_TURNS_BEFORE_COMPRESS:
        return compress_history(messages)  # See compression example above
    return messages
```

### Tool outputs consuming too many tokens

**Cause**: Tools returning raw, unfiltered data.
**Fix**: Enforce output size limits in tool wrappers; offload large results to files.

```python
def bounded_tool_output(raw_output: str, max_chars: int = 2000) -> str:
    if len(raw_output) <= max_chars:
        return raw_output
    truncated = raw_output[:max_chars]
    ref = offload_to_file(raw_output, "overflow_output")
    return f"{truncated}\n...[truncated] Full output at: {ref}"
```

### Multi-agent system producing inconsistent results

**Cause**: Agents sharing full context, causing context clash or contradictory states.
**Fix**: Use the orchestrator pattern — each agent receives only the context slice relevant to its role.

### LLM judge scores seem biased

**Cause**: Position bias in pairwise comparisons (first response rated higher).
**Fix**: Always run both orderings and compare results (see `pairwise_compare` above). Use rubric-based direct scoring as an alternative.

---

## Examples in the Repo

| Example | What It Demonstrates |
|---------|----------------------|
| `examples/digital-brain-skill/` | 6-module personal OS skill, progressive disclosure, append-only JSONL memory |
| `examples/x-to-book-system/` | Full multi-agent pipeline: monitor → synthesize → publish |
| `examples/llm-as-judge-skills/` | TypeScript LLM evaluation tools with 19 passing tests |
| `examples/book-sft-pipeline/` | LoRA fine-tuning pipeline, $2 total cost, style transfer validation |

---

## Key References

- [Context Engineering Fundamentals](skills/context-fundamentals/) — start here
- [Meta Context Engineering via Agentic Skill Evolution](https://arxiv.org/pdf/2601.21557) — academic citation of this repo
- [Cursor Plugin Directory](https://cursor.directory/plugins/context-engineering)
- [Open Plugins Standard](https://open-plugins.com)
```
