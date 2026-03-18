```markdown
---
name: agent-skills-context-engineering
description: Comprehensive collection of Agent Skills for context engineering, multi-agent architectures, memory systems, and production agent systems using Claude Code, Cursor, and other AI platforms.
triggers:
  - "context engineering for agents"
  - "build multi-agent system"
  - "install agent skills claude code"
  - "context window management"
  - "agent memory architecture"
  - "optimize agent context"
  - "implement BDI mental states"
  - "design agent evaluation framework"
---

# Agent Skills for Context Engineering

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A comprehensive, open collection of Agent Skills focused on context engineering — the discipline of curating what enters an LLM's context window to maximize agent effectiveness. Covers foundational context mechanics, multi-agent architectures, memory systems, tool design, evaluation, and cognitive modeling.

## What This Project Does

Context engineering is about managing the **holistic set of tokens** that enter a model's attention budget: system prompts, tool definitions, retrieved documents, message history, and tool outputs. This repository provides structured, installable skills that teach AI coding agents these principles across any platform.

Key problems addressed:
- **Lost-in-the-middle**: Models degrade when relevant content is buried in long contexts
- **Context poisoning/distraction**: Irrelevant tokens degrade reasoning quality
- **Attention scarcity**: More tokens ≠ better outcomes; fewer high-signal tokens do
- **Multi-agent coordination**: How agents hand off context without loss

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

Listed on [Cursor Plugin Directory](https://cursor.directory/plugins/context-engineering). Install via the Cursor plugin panel or reference `.plugin/plugin.json` directly.

### Manual / Custom Agent

Clone and reference skill files directly:

```bash
git clone https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering.git
```

Load skill content from `skills/<skill-name>/SKILL.md` into your agent's system prompt or context.

## Plugin Bundles

| Plugin | Skills Included |
|--------|-----------------|
| `context-engineering-fundamentals` | context-fundamentals, context-degradation, context-compression, context-optimization |
| `agent-architecture` | multi-agent-patterns, memory-systems, tool-design, filesystem-context, hosted-agents |
| `agent-evaluation` | evaluation, advanced-evaluation |
| `agent-development` | project-development |
| `cognitive-architecture` | bdi-mental-states |

## Repository Structure

```
Agent-Skills-for-Context-Engineering/
├── .plugin/
│   └── plugin.json              # Open Plugins manifest
├── skills/
│   ├── context-fundamentals/    # Context anatomy, token budgets
│   ├── context-degradation/     # Failure modes and diagnostics
│   ├── context-compression/     # Compression and summarization
│   ├── context-optimization/    # Caching, masking, compaction
│   ├── multi-agent-patterns/    # Orchestrator, peer, hierarchical
│   ├── memory-systems/          # Short/long-term, graph memory
│   ├── tool-design/             # Effective tool construction
│   ├── filesystem-context/      # File-based context offloading
│   ├── hosted-agents/           # Sandboxed background agents
│   ├── evaluation/              # Agent evaluation frameworks
│   ├── advanced-evaluation/     # LLM-as-a-Judge techniques
│   ├── project-development/     # LLM project methodology
│   └── bdi-mental-states/       # BDI cognitive architecture
└── examples/
    ├── digital-brain-skill/     # Personal OS for founders
    ├── x-to-book-system/        # Multi-agent X→book pipeline
    ├── llm-as-judge-skills/     # TypeScript evaluation tools
    └── book-sft-pipeline/       # Style transfer fine-tuning
```

## Core Concepts

### Context Window Anatomy

```python
# The five components competing for attention budget
context = {
    "system_prompt": "...",          # Role, instructions, constraints
    "tool_definitions": [...],       # Available tools and schemas
    "retrieved_documents": [...],    # RAG results, memory lookups
    "message_history": [...],        # Conversation turns
    "tool_outputs": [...],           # Results from tool calls
}

# Token budget allocation example
TOTAL_BUDGET = 128_000  # tokens
budget = {
    "system_prompt":      2_000,   # 1.6%  — keep tight
    "tool_definitions":   5_000,   # 3.9%  — prune unused tools
    "retrieved_documents":40_000,  # 31%   — highest ROI
    "message_history":   70_000,   # 55%   — compress aggressively
    "tool_outputs":      11_000,   # 8.5%  — offload to filesystem
}
```

### Context Degradation Patterns

```python
# Pattern 1: Lost-in-the-middle
# Critical information placed in the center of a long context
# degrades recall significantly. Always place key info at edges.

def order_context_for_attention(documents: list[str], query: str) -> list[str]:
    """Place most relevant documents first and last."""
    scored = rank_by_relevance(documents, query)
    n = len(scored)
    ordered = [None] * n
    # High relevance → positions 0 and -1
    for i, doc in enumerate(scored):
        if i % 2 == 0:
            ordered[i // 2] = doc          # fill from front
        else:
            ordered[n - 1 - i // 2] = doc  # fill from back
    return ordered

# Pattern 2: Context poisoning
# Contradictory or stale information causes unpredictable behavior
def validate_context_consistency(facts: list[dict]) -> list[dict]:
    """Remove contradicting or outdated facts before injection."""
    seen_keys = {}
    clean = []
    for fact in sorted(facts, key=lambda f: f["timestamp"], reverse=True):
        key = fact["subject"] + fact["predicate"]
        if key not in seen_keys:
            seen_keys[key] = True
            clean.append(fact)
    return clean
```

### Context Compression

```python
import anthropic

client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var

def compress_conversation(
    messages: list[dict],
    keep_last_n: int = 10,
    model: str = "claude-opus-4-5",
) -> list[dict]:
    """
    Compress long conversation history into a summary + recent tail.
    Preserves decisions, outcomes, and key entities.
    """
    if len(messages) <= keep_last_n:
        return messages

    to_compress = messages[:-keep_last_n]
    recent = messages[-keep_last_n:]

    summary_prompt = f"""Summarize this conversation segment.
Preserve: decisions made, key entities, open questions, errors encountered.
Discard: pleasantries, repetition, superseded plans.

Conversation:
{format_messages(to_compress)}
"""

    response = client.messages.create(
        model=model,
        max_tokens=1024,
        messages=[{"role": "user", "content": summary_prompt}],
    )

    summary_message = {
        "role": "assistant",
        "content": f"[COMPRESSED HISTORY]\n{response.content[0].text}",
    }

    return [summary_message] + recent


def format_messages(messages: list[dict]) -> str:
    return "\n".join(
        f"{m['role'].upper()}: {m['content']}" for m in messages
    )
```

### Multi-Agent Patterns

```python
# Orchestrator pattern — one agent routes, subagents execute
class OrchestratorAgent:
    def __init__(self, subagents: dict[str, "SubAgent"]):
        self.subagents = subagents
        self.client = anthropic.Anthropic()

    def route(self, task: str) -> str:
        """Determine which subagent handles this task."""
        routing_prompt = f"""Given this task, which specialist should handle it?
Specialists: {list(self.subagents.keys())}
Task: {task}
Reply with only the specialist name."""

        response = self.client.messages.create(
            model="claude-opus-4-5",
            max_tokens=50,
            system="You are a routing agent. Reply with only the specialist name.",
            messages=[{"role": "user", "content": routing_prompt}],
        )
        return response.content[0].text.strip()

    def execute(self, task: str) -> str:
        specialist = self.route(task)
        if specialist not in self.subagents:
            raise ValueError(f"Unknown specialist: {specialist}")
        # Pass minimal context — only what the subagent needs
        return self.subagents[specialist].run(task)


# Context handoff — pass structured summaries, not raw history
def create_handoff_context(completed_work: dict) -> str:
    """Minimal handoff context between agents."""
    return f"""AGENT HANDOFF
Task: {completed_work['task']}
Status: {completed_work['status']}
Key Outputs: {completed_work['outputs']}
Open Questions: {completed_work.get('open_questions', 'None')}
Next Agent Should: {completed_work['next_steps']}
"""
```

### Memory Systems

```python
import json
from pathlib import Path
from datetime import datetime

# Append-only JSONL memory — agent-friendly, auditable
class AgentMemory:
    def __init__(self, path: str = "agent_memory.jsonl"):
        self.path = Path(path)
        # Schema declaration as first line
        if not self.path.exists():
            self.path.write_text(
                json.dumps({"_schema": "v1", "fields": ["ts", "type", "key", "value"]}) + "\n"
            )

    def remember(self, memory_type: str, key: str, value: str) -> None:
        entry = {
            "ts": datetime.utcnow().isoformat(),
            "type": memory_type,   # "fact" | "decision" | "entity" | "error"
            "key": key,
            "value": value,
        }
        with self.path.open("a") as f:
            f.write(json.dumps(entry) + "\n")

    def recall(self, memory_type: str | None = None, limit: int = 50) -> list[dict]:
        entries = []
        with self.path.open() as f:
            for line in f:
                entry = json.loads(line)
                if "_schema" in entry:
                    continue
                if memory_type is None or entry["type"] == memory_type:
                    entries.append(entry)
        return entries[-limit:]  # most recent N

    def recall_as_context(self, memory_type: str | None = None) -> str:
        entries = self.recall(memory_type)
        if not entries:
            return "No relevant memories."
        lines = [f"[{e['ts']}] {e['type']}/{e['key']}: {e['value']}" for e in entries]
        return "\n".join(lines)


# Usage
memory = AgentMemory()
memory.remember("decision", "database_choice", "PostgreSQL — chosen for JSONB support")
memory.remember("entity", "user_id_format", "UUID v4, stored as TEXT")

# Inject into agent context
context = f"""AGENT MEMORY
{memory.recall_as_context()}
---
"""
```

### Tool Design Principles

```python
# Good tool: single responsibility, structured output, error info included
def search_codebase(
    query: str,
    file_pattern: str = "**/*.py",
    max_results: int = 10,
) -> dict:
    """
    Search codebase for relevant code.

    Returns structured results an agent can parse without hallucination.
    Always include metadata — agents need to know WHERE results came from.
    """
    import glob, re

    results = []
    for filepath in glob.glob(file_pattern, recursive=True):
        try:
            content = Path(filepath).read_text()
            if query.lower() in content.lower():
                # Find line numbers for precise context
                lines = content.splitlines()
                matches = [
                    {"line": i + 1, "text": line}
                    for i, line in enumerate(lines)
                    if query.lower() in line.lower()
                ]
                results.append({
                    "file": filepath,
                    "match_count": len(matches),
                    "matches": matches[:3],  # Top 3 per file
                })
        except (UnicodeDecodeError, PermissionError):
            pass

    return {
        "query": query,
        "total_files_matched": len(results),
        "results": results[:max_results],
        "truncated": len(results) > max_results,
    }


# Tool output offloading — don't bloat context with large outputs
def run_with_file_output(tool_fn, args: dict, output_path: str) -> str:
    """
    Run a tool and write output to file instead of returning to context.
    Returns a file reference the agent can selectively read.
    """
    result = tool_fn(**args)
    Path(output_path).write_text(json.dumps(result, indent=2))
    return f"[OUTPUT SAVED: {output_path}] — {len(str(result))} chars. Read with read_file('{output_path}')."
```

### LLM-as-Judge Evaluation

```python
import anthropic
from enum import Enum

client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY

class JudgeVerdict(Enum):
    A_BETTER = "A"
    B_BETTER = "B"
    TIE = "TIE"

def pairwise_judge(
    prompt: str,
    response_a: str,
    response_b: str,
    criteria: list[str],
    model: str = "claude-opus-4-5",
) -> dict:
    """
    Compare two responses with position bias mitigation.
    Runs A/B and B/A then averages to cancel order effects.
    """

    def single_comparison(first: str, second: str) -> str:
        criteria_text = "\n".join(f"- {c}" for c in criteria)
        judge_prompt = f"""Compare these two responses to the prompt below.

Prompt: {prompt}

Response 1:
{first}

Response 2:
{second}

Criteria:
{criteria_text}

Which response better satisfies the criteria?
Reply with exactly one of: RESPONSE_1, RESPONSE_2, TIE
Then on a new line explain in 1-2 sentences."""

        resp = client.messages.create(
            model=model,
            max_tokens=256,
            system="You are an impartial evaluator. Be concise and consistent.",
            messages=[{"role": "user", "content": judge_prompt}],
        )
        return resp.content[0].text.strip()

    # Run both orderings to mitigate position bias
    ab_result = single_comparison(response_a, response_b)
    ba_result = single_comparison(response_b, response_a)

    # Normalize: in ba_result, "RESPONSE_1" means B won
    def normalize(result: str, flipped: bool) -> JudgeVerdict:
        first_line = result.splitlines()[0]
        if "TIE" in first_line:
            return JudgeVerdict.TIE
        if "RESPONSE_1" in first_line:
            return JudgeVerdict.B_BETTER if flipped else JudgeVerdict.A_BETTER
        return JudgeVerdict.A_BETTER if flipped else JudgeVerdict.B_BETTER

    ab_verdict = normalize(ab_result, flipped=False)
    ba_verdict = normalize(ba_result, flipped=True)

    if ab_verdict == ba_verdict:
        final = ab_verdict
        confidence = "high"
    else:
        final = JudgeVerdict.TIE  # Disagreement → tie
        confidence = "low"

    return {
        "verdict": final.value,
        "confidence": confidence,
        "ab_result": ab_result,
        "ba_result": ba_result,
    }
```

### BDI Mental States

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass
class Belief:
    subject: str
    predicate: str
    object_: Any
    confidence: float = 1.0
    source: str = "observation"

@dataclass
class Desire:
    goal: str
    priority: float  # 0.0 - 1.0
    conditions: list[str] = field(default_factory=list)

@dataclass
class Intention:
    action_plan: list[str]
    committed_to: str  # which desire this serves
    status: str = "pending"  # pending | active | complete | abandoned

class BDIAgent:
    def __init__(self):
        self.beliefs: list[Belief] = []
        self.desires: list[Desire] = []
        self.intentions: list[Intention] = []

    def perceive(self, rdf_triples: list[tuple]) -> None:
        """Convert RDF context into beliefs."""
        for subject, predicate, obj in rdf_triples:
            self.beliefs.append(Belief(
                subject=subject,
                predicate=predicate,
                object_=obj,
            ))

    def deliberate(self) -> Desire | None:
        """Select highest-priority achievable desire."""
        achievable = [
            d for d in self.desires
            if self._conditions_met(d.conditions)
        ]
        if not achievable:
            return None
        return max(achievable, key=lambda d: d.priority)

    def plan(self, desire: Desire) -> Intention:
        """Generate action plan for a desire."""
        # In production: call LLM to generate plan
        steps = [f"Execute step for: {desire.goal}"]
        intention = Intention(
            action_plan=steps,
            committed_to=desire.goal,
        )
        self.intentions.append(intention)
        return intention

    def _conditions_met(self, conditions: list[str]) -> bool:
        belief_strings = {
            f"{b.subject}:{b.predicate}:{b.object_}" for b in self.beliefs
        }
        return all(c in belief_strings for c in conditions)

    def as_context_block(self) -> str:
        """Serialize mental state for injection into LLM context."""
        beliefs_text = "\n".join(
            f"  - {b.subject} {b.predicate} {b.object_} (conf={b.confidence})"
            for b in self.beliefs[-10:]
        )
        desires_text = "\n".join(
            f"  - [{b.priority:.1f}] {b.goal}" for b in self.desires
        )
        intentions_text = "\n".join(
            f"  - {i.committed_to}: {i.status}" for i in self.intentions
        )
        return f"""BDI MENTAL STATE
Beliefs (recent):
{beliefs_text}
Desires:
{desires_text}
Intentions:
{intentions_text}
"""
```

### Filesystem Context Pattern

```python
from pathlib import Path
import json

# Use filesystem as infinite context extension
class FilesystemContext:
    def __init__(self, workspace: str = ".agent_workspace"):
        self.workspace = Path(workspace)
        self.workspace.mkdir(exist_ok=True)

    def offload(self, key: str, data: Any) -> str:
        """Write large data to file, return reference string for context."""
        path = self.workspace / f"{key}.json"
        path.write_text(json.dumps(data, indent=2))
        size = len(json.dumps(data))
        return f"[FILE_REF:{key}] ({size} bytes) → {path}"

    def load(self, key: str) -> Any:
        """Load previously offloaded data."""
        path = self.workspace / f"{key}.json"
        return json.loads(path.read_text())

    def list_available(self) -> str:
        """Let agent discover what context is available."""
        files = list(self.workspace.glob("*.json"))
        if not files:
            return "No context files available."
        lines = []
        for f in files:
            size = f.stat().st_size
            lines.append(f"  - {f.stem}: {size} bytes")
        return "AVAILABLE CONTEXT FILES:\n" + "\n".join(lines)

    def write_plan(self, plan: list[str]) -> str:
        """Persist agent plan so it survives context resets."""
        return self.offload("current_plan", {"steps": plan, "current": 0})

    def tick_plan(self) -> str | None:
        """Advance to next step, return current step or None if done."""
        data = self.load("current_plan")
        idx = data["current"]
        if idx >= len(data["steps"]):
            return None
        data["current"] += 1
        self.offload("current_plan", data)
        return data["steps"][idx]
```

## Skill Trigger Reference

| Skill | Activate When User Says |
|-------|-------------------------|
| `context-fundamentals` | "explain context windows", "design agent architecture" |
| `context-degradation` | "diagnose context problems", "fix lost-in-middle", "debug agent failures" |
| `context-compression` | "compress context", "summarize conversation", "reduce token usage" |
| `context-optimization` | "optimize context", "reduce token costs", "implement KV-cache" |
| `multi-agent-patterns` | "design multi-agent system", "implement supervisor pattern" |
| `memory-systems` | "implement agent memory", "build knowledge graph", "track entities" |
| `tool-design` | "design agent tools", "reduce tool complexity", "implement MCP tools" |
| `filesystem-context` | "offload context to files", "agent scratch pad", "file-based context" |
| `hosted-agents` | "build background agent", "sandboxed execution", "multiplayer agent" |
| `evaluation` | "evaluate agent performance", "build test framework", "measure quality" |
| `advanced-evaluation` | "implement LLM-as-judge", "compare model outputs", "mitigate bias" |
| `project-development` | "start LLM project", "design batch pipeline", "evaluate task-model fit" |
| `bdi-mental-states` | "model agent mental states", "implement BDI architecture", "transform RDF to beliefs" |

## Common Patterns

### Progressive Context Loading

```python
# Only load full skill content when triggered — saves tokens on every request
class SkillLoader:
    def __init__(self, skills_dir: str = "skills"):
        self.skills_dir = Path(skills_dir)
        self._index = None

    def get_index(self) -> str:
        """Load lightweight index (names + one-line descriptions only)."""
        if self._index:
            return self._index
        skills = []
        for skill_dir in self.skills_dir.iterdir():
            readme = skill_dir / "README.md"
            if readme.exists():
                first_line = readme.read_text().splitlines()[0]
                skills.append(f"- {skill_dir.name}: {first_line}")
        self._index = "\n".join(skills)
        return self._index

    def load_skill(self, skill_name: str) -> str:
        """Load full skill content only when needed."""
        skill_file = self.skills_dir / skill_name / "SKILL.md"
        if not skill_file.exists():
            raise FileNotFoundError(f"Skill not found: {skill_name}")
        return skill_file.read_text()
```

### Token Budget Enforcement

```python
import tiktoken

def enforce_budget(
    content: str,
    max_tokens: int,
    model: str = "gpt-4o",
    strategy: str = "truncate_middle",
) -> str:
    """
    Ensure content fits within token budget.
    Strategies: truncate_end | truncate_middle | summarize
    """
    enc = tiktoken.encoding_for_model(model)
    tokens = enc.encode(content)

    if len(tokens) <= max_tokens:
        return content

    if strategy == "truncate_end":
        return enc.decode(tokens[:max_tokens])

    if strategy == "truncate_middle":
        keep = max_tokens // 2
        start = enc.decode(tokens[:keep])
        end = enc.decode(tokens[-keep:])
        return f"{start}\n\n[... {len(tokens) - max_tokens} tokens truncated ...]\n\n{end}"

    raise ValueError(f"Unknown strategy: {strategy}")
```

## Troubleshooting

### Agent loses track of earlier decisions
**Cause**: Message history too long, decisions buried in the middle.
**Fix**: Use `AgentMemory` to extract decisions into persistent JSONL; inject only the decision log at context start.

### Tool calls return too much data
**Cause**: Tool output floods the context window.
**Fix**: Use `FilesystemContext.offload()` and return file references; agent reads only what it needs.

### Multi-agent handoffs lose context
**Cause**: Raw message history passed between agents.
**Fix**: Use `create_handoff_context()` — structured summaries only, never raw history.

### LLM-as-Judge gives inconsistent verdicts
**Cause**: Position bias (model prefers whichever response appears first).
**Fix**: Use `pairwise_judge()` which runs A/B and B/A, resolves disagreements as ties.

### Agent ignores early instructions
**Cause**: Instructions in the middle of a long system prompt.
**Fix**: Place critical constraints at the **top** and **bottom** of the system prompt; use U-shaped placement.

### Context grows unbounded in long sessions
**Cause**: No compression strategy; messages accumulate.
**Fix**: Run `compress_conversation()` every N turns; keep the last 10 messages verbatim, summarize the rest.

## Environment Variables

```bash
ANTHROPIC_API_KEY=          # Required for Claude API calls
OPENAI_API_KEY=             # Optional, for OpenAI-based evaluation
AGENT_WORKSPACE_DIR=        # Optional, filesystem context directory (default: .agent_workspace)
AGENT_MEMORY_PATH=          # Optional, JSONL memory file path (default: agent_memory.jsonl)
```

## References

- [Repository](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering)
- [Cursor Plugin Directory](https://cursor.directory/plugins/context-engineering)
- [Cited in: Meta Context Engineering via Agentic Skill Evolution](https://arxiv.org/pdf/2601.21557) — Peking University (2026)
- [Open Plugins Standard](https://open-plugins.com)
```
