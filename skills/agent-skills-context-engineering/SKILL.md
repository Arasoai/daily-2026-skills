```markdown
---
name: agent-skills-context-engineering
description: Comprehensive collection of Agent Skills for context engineering, multi-agent architectures, memory systems, and production agent systems using Claude Code, Cursor, or any agent platform.
triggers:
  - "context engineering for agents"
  - "build multi-agent system"
  - "manage agent context window"
  - "implement agent memory"
  - "install agent skills claude code"
  - "optimize agent context tokens"
  - "design agent architecture"
  - "debug agent context problems"
---

# Agent Skills for Context Engineering

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A comprehensive, open collection of Agent Skills focused on context engineering principles — the discipline of curating all information entering a model's context window (system prompts, tool definitions, retrieved documents, message history, tool outputs) to maximize agent effectiveness in production systems.

## What This Project Does

This repository provides installable **Agent Skills** for AI coding agents (Claude Code, Cursor, Codex, etc.) covering:

- **Context Engineering**: Managing attention budgets, compression, and degradation prevention
- **Multi-Agent Patterns**: Orchestrator, peer-to-peer, and hierarchical architectures
- **Memory Systems**: Short-term, long-term, and graph-based memory
- **Tool Design**: Building agent-friendly tools with MCP support
- **Evaluation Frameworks**: LLM-as-Judge, pairwise comparison, rubric generation
- **Cognitive Architecture**: BDI (Beliefs, Desires, Intentions) mental state modeling
- **Hosted Agents**: Sandboxed VMs, pre-built images, multiplayer agent support

## Installation

### Claude Code (Recommended)

```bash
# Step 1: Register the marketplace
/plugin marketplace add muratcankoylan/Agent-Skills-for-Context-Engineering

# Step 2: Install individual plugin bundles
/plugin install context-engineering-fundamentals@context-engineering-marketplace
/plugin install agent-architecture@context-engineering-marketplace
/plugin install agent-evaluation@context-engineering-marketplace
/plugin install agent-development@context-engineering-marketplace
/plugin install cognitive-architecture@context-engineering-marketplace
```

### Cursor

Listed on [Cursor Plugin Directory](https://cursor.directory/plugins/context-engineering). Add via Cursor's plugin panel or reference `.plugin/plugin.json` directly.

### Manual / Custom Agent Platforms

Clone the repository and reference individual skill files:

```bash
git clone https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering.git
# Reference any skill at: skills/<skill-name>/SKILL.md
```

## Plugin Bundles & Included Skills

| Plugin | Skills Included |
|--------|----------------|
| `context-engineering-fundamentals` | context-fundamentals, context-degradation, context-compression, context-optimization |
| `agent-architecture` | multi-agent-patterns, memory-systems, tool-design, filesystem-context, hosted-agents |
| `agent-evaluation` | evaluation, advanced-evaluation |
| `agent-development` | project-development |
| `cognitive-architecture` | bdi-mental-states |

## Skill Triggers Reference

Skills activate automatically when your prompt matches these patterns:

| Skill | Activates When You Say |
|-------|----------------------|
| `context-fundamentals` | "understand context", "explain context windows", "design agent architecture" |
| `context-degradation` | "diagnose context problems", "fix lost-in-middle", "debug agent failures" |
| `context-compression` | "compress context", "summarize conversation", "reduce token usage" |
| `context-optimization` | "optimize context", "reduce token costs", "implement KV-cache" |
| `multi-agent-patterns` | "design multi-agent system", "implement supervisor pattern" |
| `memory-systems` | "implement agent memory", "build knowledge graph", "track entities" |
| `tool-design` | "design agent tools", "reduce tool complexity", "implement MCP tools" |
| `filesystem-context` | "offload context to files", "dynamic context discovery", "agent scratch pad" |
| `hosted-agents` | "build background agent", "sandboxed execution", "multiplayer agent" |
| `evaluation` | "evaluate agent performance", "build test framework", "measure quality" |
| `advanced-evaluation` | "implement LLM-as-judge", "compare model outputs", "mitigate bias" |
| `project-development` | "start LLM project", "design batch pipeline", "evaluate task-model fit" |
| `bdi-mental-states` | "model agent mental states", "implement BDI architecture", "transform RDF to beliefs" |

## Core Concepts & Code Examples

### Context Window Budget Management

The fundamental principle: find the smallest set of high-signal tokens that maximize desired outcomes.

```python
# context_budget.py — Track and enforce context budgets
from dataclasses import dataclass, field
from typing import List, Dict

@dataclass
class ContextSlot:
    name: str
    max_tokens: int
    priority: int  # 1=critical, 2=important, 3=optional
    current_tokens: int = 0

class ContextBudgetManager:
    """
    Manages token allocation across context slots.
    Enforces budget limits and prioritizes high-signal content.
    """
    TOTAL_BUDGET = 100_000  # Adjust to your model's context window

    def __init__(self):
        self.slots: Dict[str, ContextSlot] = {
            "system_prompt":   ContextSlot("system_prompt",   2_000, priority=1),
            "tool_definitions":ContextSlot("tool_definitions",3_000, priority=1),
            "retrieved_docs":  ContextSlot("retrieved_docs", 20_000, priority=2),
            "message_history": ContextSlot("message_history",60_000, priority=2),
            "tool_outputs":    ContextSlot("tool_outputs",   10_000, priority=3),
            "working_memory":  ContextSlot("working_memory",  5_000, priority=3),
        }

    def allocate(self, slot_name: str, content: str, tokenizer) -> str:
        """Allocate content to a slot, truncating if over budget."""
        slot = self.slots[slot_name]
        tokens = tokenizer.encode(content)
        if len(tokens) > slot.max_tokens:
            # Truncate and append summary marker
            tokens = tokens[:slot.max_tokens - 50]
            content = tokenizer.decode(tokens) + "\n[...truncated for context budget]"
            slot.current_tokens = slot.max_tokens
        else:
            slot.current_tokens = len(tokens)
        return content

    def get_utilization(self) -> dict:
        used = sum(s.current_tokens for s in self.slots.values())
        return {
            "used": used,
            "total": self.TOTAL_BUDGET,
            "pct": round(used / self.TOTAL_BUDGET * 100, 1),
            "by_slot": {k: v.current_tokens for k, v in self.slots.items()}
        }
```

### Context Compression Pipeline

```python
# context_compression.py — Hierarchical compression strategy
import anthropic

client = anthropic.Anthropic()  # Uses ANTHROPIC_API_KEY env var

def compress_message_history(
    messages: list[dict],
    target_tokens: int = 8_000,
    preserve_last_n: int = 6
) -> list[dict]:
    """
    Compress message history using a 3-tier strategy:
    1. Preserve the last N messages verbatim (recency bias)
    2. Summarize middle messages into a rolling summary
    3. Drop oldest messages beyond summary scope
    """
    if len(messages) <= preserve_last_n:
        return messages

    recent = messages[-preserve_last_n:]
    to_compress = messages[:-preserve_last_n]

    # Build summary of older messages
    history_text = "\n".join(
        f"{m['role'].upper()}: {m['content']}" for m in to_compress
    )

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": (
                f"Summarize this conversation history, preserving:\n"
                f"- Key decisions made\n- Important facts established\n"
                f"- Current task state\n- Unresolved questions\n\n"
                f"History:\n{history_text}"
            )
        }]
    )
    summary = response.content[0].text

    compressed = [
        {"role": "system", "content": f"[CONVERSATION SUMMARY]\n{summary}"},
        *recent
    ]
    return compressed


def rolling_summary_agent(session_messages: list[dict]) -> list[dict]:
    """Automatically compress when history exceeds threshold."""
    COMPRESSION_THRESHOLD = 40  # messages
    if len(session_messages) > COMPRESSION_THRESHOLD:
        return compress_message_history(session_messages)
    return session_messages
```

### Multi-Agent Orchestrator Pattern

```python
# orchestrator.py — Supervisor pattern for multi-agent systems
import anthropic
import json
from typing import Callable

client = anthropic.Anthropic()

# Define specialized sub-agents as tools
AGENT_TOOLS = [
    {
        "name": "research_agent",
        "description": "Searches and synthesizes information from external sources. Use for fact-finding, web search, document retrieval.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Research query"},
                "depth": {"type": "string", "enum": ["shallow", "deep"], "default": "shallow"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "code_agent",
        "description": "Writes, tests, and debugs code. Use for implementation tasks.",
        "input_schema": {
            "type": "object",
            "properties": {
                "task": {"type": "string", "description": "Coding task description"},
                "language": {"type": "string", "description": "Programming language"}
            },
            "required": ["task"]
        }
    },
    {
        "name": "critic_agent",
        "description": "Reviews and evaluates work quality. Use for validation and quality checks.",
        "input_schema": {
            "type": "object",
            "properties": {
                "content": {"type": "string", "description": "Content to review"},
                "criteria": {"type": "array", "items": {"type": "string"}}
            },
            "required": ["content"]
        }
    }
]


def orchestrator(
    task: str,
    agent_registry: dict[str, Callable],
    max_iterations: int = 10
) -> str:
    """
    Supervisor orchestrator that delegates to specialized sub-agents.
    Maintains minimal context — passes only task-relevant information.
    """
    messages = [{"role": "user", "content": task}]
    system = (
        "You are an orchestrator. Decompose complex tasks and delegate to specialized agents. "
        "Always use the most specific agent available. Return FINAL ANSWER: <result> when done."
    )

    for iteration in range(max_iterations):
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=system,
            tools=AGENT_TOOLS,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            # Extract final answer
            for block in response.content:
                if hasattr(block, "text") and "FINAL ANSWER:" in block.text:
                    return block.text.split("FINAL ANSWER:")[-1].strip()
            break

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})
            tool_results = []

            for block in response.content:
                if block.type == "tool_use":
                    agent_fn = agent_registry.get(block.name)
                    if agent_fn:
                        result = agent_fn(**block.input)
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": str(result)
                        })

            messages.append({"role": "user", "content": tool_results})

    return "Orchestration complete — max iterations reached."
```

### Memory System Architecture

```python
# memory_system.py — Three-tier memory for persistent agents
import json
import time
from pathlib import Path
from collections import deque

class AgentMemorySystem:
    """
    Three-tier memory following the skills architecture:
    - Working Memory: In-context, current task state (fast, volatile)
    - Episodic Memory: Recent interactions, append-only JSONL (medium persistence)
    - Semantic Memory: Distilled facts and knowledge (long-term, indexed)
    """

    def __init__(self, agent_id: str, base_dir: str = ".agent_memory"):
        self.agent_id = agent_id
        self.base = Path(base_dir) / agent_id
        self.base.mkdir(parents=True, exist_ok=True)

        # Working memory — stays in context window
        self.working: deque = deque(maxlen=20)

        # Episodic memory — append-only JSONL file
        self.episodic_path = self.base / "episodes.jsonl"

        # Semantic memory — distilled facts as JSON
        self.semantic_path = self.base / "semantic.json"
        self.semantic: dict = self._load_semantic()

    def remember(self, content: str, memory_type: str = "episodic", metadata: dict = None):
        """Store a memory with automatic routing by type."""
        entry = {
            "timestamp": time.time(),
            "agent_id": self.agent_id,
            "content": content,
            "type": memory_type,
            "metadata": metadata or {}
        }

        if memory_type == "working":
            self.working.append(entry)

        elif memory_type == "episodic":
            # Schema-first line for agent-friendly parsing
            with open(self.episodic_path, "a") as f:
                f.write(json.dumps(entry) + "\n")

        elif memory_type == "semantic":
            key = metadata.get("key", f"fact_{int(time.time())}")
            self.semantic[key] = {"content": content, "updated": time.time()}
            self._save_semantic()

    def recall(self, memory_type: str = "working", last_n: int = 10) -> list[dict]:
        """Retrieve memories with recency bias."""
        if memory_type == "working":
            return list(self.working)[-last_n:]

        elif memory_type == "episodic":
            if not self.episodic_path.exists():
                return []
            lines = self.episodic_path.read_text().strip().split("\n")
            entries = [json.loads(l) for l in lines if l.strip()]
            return entries[-last_n:]

        elif memory_type == "semantic":
            return list(self.semantic.values())

        return []

    def get_context_snapshot(self) -> str:
        """
        Build a minimal context injection from memory.
        Returns the most relevant memories as a formatted string.
        """
        working = self.recall("working", last_n=5)
        semantic_facts = self.recall("semantic")

        parts = []
        if semantic_facts:
            facts = "\n".join(f"- {f['content']}" for f in semantic_facts[:10])
            parts.append(f"[Known Facts]\n{facts}")
        if working:
            recent = "\n".join(f"- {m['content']}" for m in working)
            parts.append(f"[Working Context]\n{recent}")

        return "\n\n".join(parts)

    def _load_semantic(self) -> dict:
        if self.semantic_path.exists():
            return json.loads(self.semantic_path.read_text())
        return {}

    def _save_semantic(self):
        self.semantic_path.write_text(json.dumps(self.semantic, indent=2))


# Usage
memory = AgentMemorySystem("my-agent")
memory.remember("User prefers Python over TypeScript", "semantic", {"key": "lang_preference"})
memory.remember("Currently working on auth module", "working")
context = memory.get_context_snapshot()
```

### LLM-as-Judge Evaluation

```python
# evaluator.py — Production LLM-as-Judge implementation
import anthropic
from dataclasses import dataclass

client = anthropic.Anthropic()

@dataclass
class EvaluationResult:
    score: float          # 0.0 - 1.0
    reasoning: str
    criteria_scores: dict[str, float]
    passed: bool          # score >= threshold

def direct_score(
    response: str,
    criteria: list[dict],   # [{"name": str, "weight": float, "description": str}]
    threshold: float = 0.7,
    rubric: str = ""
) -> EvaluationResult:
    """
    Score a response against weighted criteria using LLM-as-Judge.
    Mitigates positional bias by using structured scoring.
    """
    criteria_text = "\n".join(
        f"{i+1}. {c['name']} (weight: {c['weight']}): {c['description']}"
        for i, c in enumerate(criteria)
    )

    prompt = f"""Evaluate this response against the criteria below.
For each criterion, provide a score from 0.0 to 1.0 and brief reasoning.
{f'Rubric: {rubric}' if rubric else ''}

CRITERIA:
{criteria_text}

RESPONSE TO EVALUATE:
{response}

Respond in JSON:
{{
  "criteria_scores": {{"criterion_name": {{"score": 0.0-1.0, "reasoning": "..."}}}},
  "overall_reasoning": "..."
}}"""

    eval_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )

    import json, re
    text = eval_response.content[0].text
    json_match = re.search(r'\{.*\}', text, re.DOTALL)
    result_data = json.loads(json_match.group()) if json_match else {}

    criteria_scores = {}
    weighted_total = 0.0
    total_weight = sum(c["weight"] for c in criteria)

    for c in criteria:
        name = c["name"]
        score_data = result_data.get("criteria_scores", {}).get(name, {"score": 0.5})
        score = score_data.get("score", 0.5) if isinstance(score_data, dict) else 0.5
        criteria_scores[name] = score
        weighted_total += score * (c["weight"] / total_weight)

    return EvaluationResult(
        score=round(weighted_total, 3),
        reasoning=result_data.get("overall_reasoning", ""),
        criteria_scores=criteria_scores,
        passed=weighted_total >= threshold
    )


def pairwise_compare(
    response_a: str,
    response_b: str,
    task: str,
    swap_and_average: bool = True  # Mitigate position bias
) -> dict:
    """Compare two responses, optionally swapping positions to reduce bias."""

    def _compare(first: str, second: str) -> str:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=512,
            messages=[{"role": "user", "content": (
                f"Task: {task}\n\nResponse A:\n{first}\n\nResponse B:\n{second}\n\n"
                "Which response better completes the task? Reply with 'A', 'B', or 'TIE' "
                "and one sentence of reasoning."
            )}]
        )
        return response.content[0].text

    result1 = _compare(response_a, response_b)
    if not swap_and_average:
        return {"winner": result1}

    result2 = _compare(response_b, response_a)  # Swapped
    # Normalize swapped result (A in position 2 = B in result2)
    normalized2 = result2.replace(" A", " __B__").replace(" B", " A").replace(" __B__", " B")

    return {
        "forward": result1,
        "backward_normalized": normalized2,
        "consensus": result1 if result1 == normalized2 else "INCONCLUSIVE"
    }
```

### Filesystem Context Pattern

```python
# filesystem_context.py — Use filesystem for dynamic context discovery
import json
from pathlib import Path
from datetime import datetime

class FilesystemContextManager:
    """
    Offload large context to files; inject only summaries into context window.
    Follows the filesystem-context skill pattern.
    """

    def __init__(self, workspace: str = ".agent_workspace"):
        self.workspace = Path(workspace)
        self.workspace.mkdir(exist_ok=True)
        (self.workspace / "plans").mkdir(exist_ok=True)
        (self.workspace / "outputs").mkdir(exist_ok=True)
        (self.workspace / "scratch").mkdir(exist_ok=True)

    def save_plan(self, plan_id: str, steps: list[dict]) -> Path:
        """Persist multi-step plan to filesystem instead of context."""
        plan_path = self.workspace / "plans" / f"{plan_id}.json"
        plan_data = {
            "id": plan_id,
            "created": datetime.now().isoformat(),
            "steps": steps,
            "status": {step["id"]: "pending" for step in steps}
        }
        plan_path.write_text(json.dumps(plan_data, indent=2))
        return plan_path

    def update_plan_step(self, plan_id: str, step_id: str, status: str, output: str = ""):
        """Update a step status without holding full plan in context."""
        plan_path = self.workspace / "plans" / f"{plan_id}.json"
        plan = json.loads(plan_path.read_text())
        plan["status"][step_id] = status
        if output:
            output_file = self.workspace / "outputs" / f"{plan_id}_{step_id}.txt"
            output_file.write_text(output)
            plan["output_refs"] = plan.get("output_refs", {})
            plan["output_refs"][step_id] = str(output_file)
        plan_path.write_text(json.dumps(plan, indent=2))

    def get_plan_summary(self, plan_id: str) -> str:
        """
        Return minimal plan summary for context injection.
        Load full outputs only when needed.
        """
        plan_path = self.workspace / "plans" / f"{plan_id}.json"
        plan = json.loads(plan_path.read_text())
        status_summary = "\n".join(
            f"  [{v.upper()}] {k}" for k, v in plan["status"].items()
        )
        pending = [k for k, v in plan["status"].items() if v == "pending"]
        return (
            f"Plan: {plan_id}\n"
            f"Steps:\n{status_summary}\n"
            f"Next: {pending[0] if pending else 'COMPLETE'}"
        )

    def scratch(self, key: str, content: str = None) -> str | None:
        """Agent scratch pad — read/write without polluting context."""
        path = self.workspace / "scratch" / f"{key}.txt"
        if content is not None:
            path.write_text(content)
            return None
        return path.read_text() if path.exists() else None


# Usage pattern
ctx = FilesystemContextManager()
plan_path = ctx.save_plan("feature-auth", [
    {"id": "design", "description": "Design auth schema"},
    {"id": "implement", "description": "Write auth code"},
    {"id": "test", "description": "Write and run tests"},
])

# During execution — only inject summary, not full plan
summary = ctx.get_plan_summary("feature-auth")
print(summary)
# Plan: feature-auth
# Steps:
#   [PENDING] design
#   [PENDING] implement
#   [PENDING] test
# Next: design
```

## Common Patterns

### Pattern 1: Progressive Context Loading

Load only what's needed, when it's needed:

```python
class ProgressiveContextLoader:
    """Load context in three levels: names → summaries → full content."""

    def __init__(self, skills_dir: str = "skills"):
        self.skills_dir = Path(skills_dir)

    def level_1_names(self) -> list[str]:
        """Startup: load only skill names (minimal tokens)."""
        return [d.name for d in self.skills_dir.iterdir() if d.is_dir()]

    def level_2_descriptions(self) -> dict[str, str]:
        """On demand: load descriptions when task context is known."""
        result = {}
        for skill_dir in self.skills_dir.iterdir():
            readme = skill_dir / "README.md"
            if readme.exists():
                # Extract first non-heading line as description
                for line in readme.read_text().splitlines():
                    if line.strip() and not line.startswith("#"):
                        result[skill_dir.name] = line.strip()
                        break
        return result

    def level_3_full(self, skill_name: str) -> str:
        """Activation: load full skill content only when triggered."""
        skill_file = self.skills_dir / skill_name / "SKILL.md"
        if skill_file.exists():
            return skill_file.read_text()
        return f"Skill '{skill_name}' not found."
```

### Pattern 2: Context Poisoning Prevention

```python
def sanitize_tool_output(output: str, max_tokens: int = 2000) -> str:
    """
    Prevent context poisoning from verbose tool outputs.
    Truncate, extract key info, and mark truncation clearly.
    """
    # Remove common noise patterns
    import re
    noise_patterns = [
        r'\x1b\[[0-9;]*m',          # ANSI color codes
        r'^\s*at .*\(.*\)\s*$',     # Stack trace lines
        r'^\s*\.\.\.\s*$',          # Ellipsis-only lines
    ]
    for pattern in noise_patterns:
        output = re.sub(pattern, '', output, flags=re.MULTILINE)

    lines = output.strip().splitlines()

    # Rough token estimate: 1 token ≈ 4 chars
    char_budget = max_tokens * 4
    if len(output) <= char_budget:
        return output

    # Keep first 60% and last 20% (avoid lost-in-middle)
    head_budget = int(char_budget * 0.6)
    tail_budget = int(char_budget * 0.2)
    head = output[:head_budget]
    tail = output[-tail_budget:] if tail_budget > 0 else ""
    omitted = len(output) - head_budget - tail_budget

    return (
        f"{head}\n\n[... {omitted} characters omitted for context budget ...]\n\n{tail}"
    )
```

### Pattern 3: BDI Mental State Modeling

```python
# bdi_agent.py — Beliefs, Desires, Intentions cognitive architecture
from dataclasses import dataclass, field
from typing import Any

@dataclass
class Belief:
    subject: str
    predicate: str
    object: Any
    confidence: float = 1.0
    source: str = "observation"

@dataclass
class Desire:
    goal: str
    priority: int  # 1=highest
    conditions: list[str] = field(default_factory=list)

@dataclass
class Intention:
    action: str
    rationale: str
    derived_from: str  # desire.goal that triggered this

class BDIAgent:
    """Deliberative agent using BDI cognitive architecture."""

    def __init__(self, name: str):
        self.name = name
        self.beliefs: list[Belief] = []
        self.desires: list[Desire] = []
        self.intentions: list[Intention] = []

    def perceive(self, observation: dict):
        """Update beliefs from environment observation."""
        for key, value in observation.items():
            self.beliefs.append(Belief(
                subject=self.name,
                predicate=key,
                object=value,
                source="perception"
            ))

    def deliberate(self) -> list[Intention]:
        """
        Deliberation cycle: filter desires by belief state,
        generate intentions for achievable goals.
        """
        self.intentions = []
        active_desires = sorted(self.desires, key=lambda d: d.priority)

        for desire in active_desires:
            # Check if conditions are met by current beliefs
            belief_facts = {f"{b.predicate}={b.object}" for b in self.beliefs}
            conditions_met = all(c in belief_facts for c in desire.conditions)

            if conditions_met or not desire.conditions:
                intention = Intention(
                    action=f"pursue_{desire.goal.replace(' ', '_')}",
                    rationale=f"Goal '{desire.goal}' is achievable given current beliefs",
                    derived_from=desire.goal
                )
                self.intentions.append(intention)

        return self.intentions

    def get_context_for_llm(self) -> str:
        """Format BDI state as minimal context for LLM reasoning."""
        beliefs_text = "\n".join(
            f"  - {b.predicate}: {b.object} (confidence: {b.confidence})"
            for b in self.beliefs[-10:]  # Last 10 beliefs only
        )
        intentions_text = "\n".join(
            f"  - {i.action}: {i.rationale}"
            for i in self.intentions[:5]  # Top 5 intentions
        )
        return (
            f"[Agent Beliefs]\n{beliefs_text}\n\n"
            f"[Current Intentions]\n{intentions_text}"
        )
```

## Troubleshooting

### Plugin Not Activating in Claude Code

```bash
# Verify marketplace registration
/plugin list marketplaces

# Reinstall specific plugin
/plugin uninstall context-engineering-fundamentals
/plugin install context-engineering-fundamentals@context-engineering-marketplace

# Check plugin status
/plugin list installed
```

### "Lost-in-the-Middle" Context Degradation

**Symptom**: Agent ignores important information placed in the middle of long contexts.

**Fix**: Restructure context with critical information at start and end:

```python
def anti_lost_in_middle_context(
    critical_instructions: str,
    background_docs: list[str],
    immediate_task: str
) -> str:
    """
    U-shaped context structure to combat attention degradation.
    Place high-priority content at edges, background in middle.
    """
    return (
        f"[CRITICAL - READ FIRST]\n{critical_instructions}\n\n"
        f"[BACKGROUND CONTEXT]\n" +
        "\
