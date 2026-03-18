```markdown
---
name: agent-skills-context-engineering
description: Comprehensive collection of Agent Skills for context engineering, multi-agent architectures, memory systems, and production agent systems using Claude Code, Cursor, and other AI coding platforms.
triggers:
  - "context engineering for agents"
  - "build multi-agent system"
  - "implement agent memory"
  - "optimize context window"
  - "install agent skills claude code"
  - "design agent architecture"
  - "debug context problems"
  - "evaluate agent performance"
---

# Agent Skills for Context Engineering

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A comprehensive, open collection of Agent Skills focused on context engineering principles for building production-grade AI agent systems. Context engineering is the discipline of managing what enters the model's context window — system prompts, tool definitions, retrieved documents, message history, and tool outputs — to maximize agent effectiveness within limited attention budgets.

## What This Project Does

This repository provides installable "skills" (structured markdown files with YAML frontmatter) that AI coding agents load to gain expertise in:

- **Context fundamentals**: anatomy of context windows, attention mechanics, token budgeting
- **Context failure patterns**: lost-in-middle, poisoning, distraction, attention scarcity
- **Multi-agent architectures**: orchestrator, peer-to-peer, hierarchical patterns
- **Memory systems**: short-term, long-term, graph-based memory
- **Tool design**: building agent-effective tools and MCP integrations
- **Evaluation frameworks**: LLM-as-Judge, pairwise comparison, rubric generation
- **Cognitive architectures**: BDI mental states, RDF context transformation
- **Hosted agents**: sandboxed VMs, multiplayer support, background coding agents

## Installation

### Claude Code (Primary Platform)

**Step 1: Register the marketplace**

```bash
/plugin marketplace add muratcankoylan/Agent-Skills-for-Context-Engineering
```

**Step 2: Install plugin bundles**

```bash
# Foundational context engineering skills
/plugin install context-engineering-fundamentals@context-engineering-marketplace

# Agent architecture patterns
/plugin install agent-architecture@context-engineering-marketplace

# Evaluation frameworks
/plugin install agent-evaluation@context-engineering-marketplace

# Project development methodology
/plugin install agent-development@context-engineering-marketplace

# BDI cognitive architecture
/plugin install cognitive-architecture@context-engineering-marketplace
```

**Step 3: Browse interactively**

```
/plugin marketplace add muratcankoylan/Agent-Skills-for-Context-Engineering
→ Browse and install plugins
→ context-engineering-marketplace
→ Select plugin → Install now
```

### Cursor

Listed on the [Cursor Plugin Directory](https://cursor.directory/plugins/context-engineering). Install via the Cursor plugin panel by searching `context-engineering`.

### Manual / Custom Agent Frameworks

Clone and reference skills directly:

```bash
git clone https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering.git
```

Each skill lives at `skills/<skill-name>/SKILL.md`. Load them into your agent's context as needed.

## Plugin Bundles

| Plugin | Skills Included |
|--------|-----------------|
| `context-engineering-fundamentals` | context-fundamentals, context-degradation, context-compression, context-optimization |
| `agent-architecture` | multi-agent-patterns, memory-systems, tool-design, filesystem-context, hosted-agents |
| `agent-evaluation` | evaluation, advanced-evaluation |
| `agent-development` | project-development |
| `cognitive-architecture` | bdi-mental-states |

## Skill Trigger Reference

Skills activate automatically when these phrases appear in conversation:

| Skill | Natural Triggers |
|-------|-----------------|
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

## Core Concepts

### Context Engineering vs Prompt Engineering

```python
# Prompt engineering: craft better instructions
system_prompt = "You are a helpful assistant. Be concise and accurate."

# Context engineering: curate ALL information entering the context window
def build_agent_context(task, history, tools, retrieved_docs):
    return {
        # Only include tools relevant to THIS task
        "tools": filter_relevant_tools(tools, task),
        # Compress history, preserving high-signal moments
        "history": compress_history(history, max_tokens=2000),
        # Rank and trim retrieved docs by relevance
        "documents": rank_and_trim(retrieved_docs, task, max_tokens=3000),
        # System prompt stays lean
        "system": minimal_system_prompt(task),
    }
```

### Progressive Disclosure Pattern

The repository itself demonstrates progressive disclosure — load only what's needed:

```python
# Level 1: Agent startup — load skill names + descriptions only
skills_index = load_file("skills/INDEX.md")  # ~500 tokens

# Level 2: Task match — load relevant skill overview
if task_requires("memory"):
    skill_overview = load_file("skills/memory-systems/SKILL.md")  # ~2000 tokens

# Level 3: Deep implementation — load full detail on demand
if implementing_graph_memory:
    full_detail = load_file("skills/memory-systems/graph-patterns.md")  # ~5000 tokens
```

## Real Code Examples

### Multi-Agent Orchestrator Pattern

```python
from typing import TypedDict, Literal
import anthropic

client = anthropic.Anthropic()

class AgentState(TypedDict):
    task: str
    subtasks: list[str]
    results: list[str]
    context_budget: int

def orchestrator(state: AgentState) -> AgentState:
    """Routes tasks to specialized subagents, manages context budget."""
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system="""You are an orchestrator. Decompose the task into subtasks.
        Return JSON: {"subtasks": ["subtask1", "subtask2"]}""",
        messages=[{"role": "user", "content": state["task"]}]
    )
    subtasks = parse_json(response.content[0].text)["subtasks"]
    return {**state, "subtasks": subtasks}

def subagent(subtask: str, context_budget: int) -> str:
    """Specialized worker with bounded context."""
    response = client.messages.create(
        model="claude-haiku-4-5",  # cheaper model for subtasks
        max_tokens=min(512, context_budget // len(subtask)),
        messages=[{"role": "user", "content": subtask}]
    )
    return response.content[0].text

def run_pipeline(task: str) -> list[str]:
    state = AgentState(task=task, subtasks=[], results=[], context_budget=8000)
    state = orchestrator(state)
    return [subagent(t, state["context_budget"]) for t in state["subtasks"]]
```

### Context Compression

```python
import anthropic

client = anthropic.Anthropic()

def compress_conversation(
    messages: list[dict],
    max_tokens: int = 2000,
    preserve_last_n: int = 3
) -> list[dict]:
    """
    Compress long conversation history while preserving recent messages.
    Implements the context-compression skill pattern.
    """
    if len(messages) <= preserve_last_n:
        return messages

    # Always preserve the most recent N messages verbatim
    recent = messages[-preserve_last_n:]
    to_compress = messages[:-preserve_last_n]

    # Summarize older messages
    summary_response = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=max_tokens,
        system="Summarize this conversation history. Preserve: decisions made, "
               "key facts established, open questions, and any explicit constraints.",
        messages=[{
            "role": "user",
            "content": "\n".join(f"{m['role']}: {m['content']}" for m in to_compress)
        }]
    )

    summary_message = {
        "role": "user",
        "content": f"[Conversation summary]\n{summary_response.content[0].text}"
    }

    return [summary_message] + recent


def estimate_tokens(text: str) -> int:
    """Rough token estimation: ~4 chars per token."""
    return len(text) // 4


def context_aware_truncate(
    documents: list[str],
    query: str,
    budget: int
) -> list[str]:
    """Select highest-relevance documents within token budget."""
    # Score documents by keyword overlap (production: use embeddings)
    query_terms = set(query.lower().split())
    scored = []
    for doc in documents:
        doc_terms = set(doc.lower().split())
        score = len(query_terms & doc_terms) / max(len(query_terms), 1)
        scored.append((score, doc))

    scored.sort(reverse=True)

    selected, used = [], 0
    for _, doc in scored:
        tokens = estimate_tokens(doc)
        if used + tokens <= budget:
            selected.append(doc)
            used += tokens
        else:
            break

    return selected
```

### Memory System Implementation

```python
import json
import time
from pathlib import Path
from dataclasses import dataclass, asdict

@dataclass
class MemoryEntry:
    id: str
    content: str
    type: str          # "fact" | "decision" | "entity" | "episode"
    importance: float  # 0.0–1.0
    timestamp: float
    tags: list[str]

class AgentMemory:
    """
    Implements the memory-systems skill pattern.
    Uses append-only JSONL for agent-friendly parsing.
    """

    def __init__(self, memory_path: str = "agent_memory.jsonl"):
        self.path = Path(memory_path)
        self._ensure_schema()

    def _ensure_schema(self):
        """Schema-first line enables fast parsing without full load."""
        if not self.path.exists():
            schema = {"_schema": "v1", "fields": list(MemoryEntry.__dataclass_fields__.keys())}
            self.path.write_text(json.dumps(schema) + "\n")

    def store(self, entry: MemoryEntry) -> None:
        """Append-only write — never mutate existing memories."""
        with self.path.open("a") as f:
            f.write(json.dumps(asdict(entry)) + "\n")

    def retrieve(
        self,
        query: str,
        max_results: int = 10,
        min_importance: float = 0.3
    ) -> list[MemoryEntry]:
        """Load and filter memories. Production: replace with vector search."""
        memories = []
        query_terms = set(query.lower().split())

        with self.path.open() as f:
            for line in f:
                data = json.loads(line)
                if "_schema" in data:
                    continue
                entry = MemoryEntry(**data)
                if entry.importance < min_importance:
                    continue
                # Score by term overlap + recency
                terms = set(entry.content.lower().split())
                overlap = len(query_terms & terms) / max(len(query_terms), 1)
                recency = 1.0 / (1.0 + (time.time() - entry.timestamp) / 86400)
                score = 0.7 * overlap + 0.3 * recency
                memories.append((score, entry))

        memories.sort(reverse=True)
        return [e for _, e in memories[:max_results]]

    def forget(self, older_than_days: float = 30, below_importance: float = 0.2):
        """Prune low-value old memories to manage context budget."""
        cutoff = time.time() - (older_than_days * 86400)
        kept = []
        with self.path.open() as f:
            for line in f:
                data = json.loads(line)
                if "_schema" in data:
                    kept.append(line)
                    continue
                entry = MemoryEntry(**data)
                if entry.timestamp > cutoff or entry.importance >= below_importance:
                    kept.append(line)
        self.path.write_text("".join(kept))
```

### LLM-as-Judge Evaluation

```python
import anthropic
from dataclasses import dataclass

client = anthropic.Anthropic()

@dataclass
class EvalResult:
    score: float       # 0.0–1.0
    reasoning: str
    criteria_scores: dict[str, float]

JUDGE_SYSTEM = """You are an impartial evaluator. Score the response on each criterion.
Return JSON: {"overall": 0.85, "reasoning": "...", "criteria": {"accuracy": 0.9, "conciseness": 0.8}}"""

def direct_score(
    prompt: str,
    response: str,
    criteria: dict[str, str],  # {"criterion_name": "description"}
    model: str = "claude-opus-4-5"
) -> EvalResult:
    """
    Implements the advanced-evaluation skill: direct scoring pattern.
    """
    criteria_text = "\n".join(f"- {k}: {v}" for k, v in criteria.items())
    eval_prompt = f"""Rate this response on the following criteria (0.0–1.0 each):
{criteria_text}

Original prompt: {prompt}

Response to evaluate:
{response}"""

    result = client.messages.create(
        model=model,
        max_tokens=512,
        system=JUDGE_SYSTEM,
        messages=[{"role": "user", "content": eval_prompt}]
    )

    data = json.loads(result.content[0].text)
    return EvalResult(
        score=data["overall"],
        reasoning=data["reasoning"],
        criteria_scores=data["criteria"]
    )

def pairwise_compare(
    prompt: str,
    response_a: str,
    response_b: str,
    model: str = "claude-opus-4-5"
) -> dict:
    """
    Pairwise comparison with position bias mitigation.
    Run A-vs-B and B-vs-A, average results.
    """
    def compare(r1: str, r2: str, label1: str, label2: str) -> dict:
        result = client.messages.create(
            model=model,
            max_tokens=256,
            system='Compare two responses. Return JSON: {"winner": "A"|"B"|"tie", "reasoning": "..."}',
            messages=[{"role": "user", "content":
                f"Prompt: {prompt}\n\n{label1}: {r1}\n\n{label2}: {r2}\n\nWhich is better?"}]
        )
        return json.loads(result.content[0].text)

    forward = compare(response_a, response_b, "A", "B")
    backward = compare(response_b, response_a, "A", "B")

    # Normalize backward result (A in backward = response_b)
    if backward["winner"] == "A":
        backward["winner"] = "B"
    elif backward["winner"] == "B":
        backward["winner"] = "A"

    if forward["winner"] == backward["winner"]:
        return {"winner": forward["winner"], "confidence": "high",
                "reasoning": forward["reasoning"]}
    return {"winner": "tie", "confidence": "low",
            "reasoning": "Position bias detected — inconsistent results"}
```

### Filesystem Context Pattern

```python
import json
from pathlib import Path
from datetime import datetime

class FilesystemContext:
    """
    Implements filesystem-context skill: offload large outputs to files,
    use filesystem as dynamic context discovery mechanism.
    """

    def __init__(self, workspace: str = ".agent_workspace"):
        self.workspace = Path(workspace)
        self.workspace.mkdir(exist_ok=True)
        (self.workspace / "plans").mkdir(exist_ok=True)
        (self.workspace / "outputs").mkdir(exist_ok=True)
        (self.workspace / "scratch").mkdir(exist_ok=True)

    def save_plan(self, plan_id: str, steps: list[dict]) -> Path:
        """Persist multi-step plan to filesystem instead of context."""
        plan_file = self.workspace / "plans" / f"{plan_id}.json"
        plan_file.write_text(json.dumps({
            "id": plan_id,
            "created": datetime.utcnow().isoformat(),
            "steps": steps,
            "completed": []
        }, indent=2))
        return plan_file

    def load_plan(self, plan_id: str) -> dict:
        plan_file = self.workspace / "plans" / f"{plan_id}.json"
        return json.loads(plan_file.read_text())

    def mark_step_complete(self, plan_id: str, step_index: int, result: str):
        plan = self.load_plan(plan_id)
        plan["completed"].append({"step": step_index, "result": result,
                                   "at": datetime.utcnow().isoformat()})
        plan_file = self.workspace / "plans" / f"{plan_id}.json"
        plan_file.write_text(json.dumps(plan, indent=2))

    def offload_output(self, key: str, content: str) -> str:
        """Save large tool output to file, return reference token for context."""
        output_file = self.workspace / "outputs" / f"{key}.txt"
        output_file.write_text(content)
        # Return a compact reference instead of full content
        return f"[FILE:{output_file}|{len(content)}chars|{estimate_tokens(content)}tokens]"

    def discover_context(self) -> dict:
        """Scan workspace to build minimal context summary for agent startup."""
        plans = list((self.workspace / "plans").glob("*.json"))
        outputs = list((self.workspace / "outputs").glob("*"))
        return {
            "active_plans": [p.stem for p in plans],
            "available_outputs": [o.name for o in outputs],
            "workspace": str(self.workspace)
        }
```

## Common Patterns

### Pattern 1: Token Budget Enforcement

```python
MAX_CONTEXT_TOKENS = 8000
TOOL_BUDGET = 2000
HISTORY_BUDGET = 3000
DOCS_BUDGET = 2000
SYSTEM_BUDGET = 1000

def build_bounded_context(system, history, tools, docs, query):
    return {
        "system": truncate_to(system, SYSTEM_BUDGET),
        "tools": select_tools(tools, query, TOOL_BUDGET),
        "history": compress_history(history, HISTORY_BUDGET),
        "documents": rank_and_trim(docs, query, DOCS_BUDGET),
    }
```

### Pattern 2: Avoid Lost-in-Middle

```python
def arrange_context_for_attention(documents: list[str]) -> list[str]:
    """
    U-shaped attention: models attend better to start and end.
    Place highest-importance content first and last.
    """
    if len(documents) <= 2:
        return documents
    by_importance = sorted(documents, key=score_importance, reverse=True)
    # Most important first, second most important last
    result = [by_importance[0]]
    result.extend(by_importance[2:])   # middle: lower importance
    result.append(by_importance[1])    # end: second most important
    return result
```

### Pattern 3: Tool Output Offloading

```python
def agent_tool_call(tool_name: str, args: dict, fs_ctx: FilesystemContext) -> str:
    raw_output = execute_tool(tool_name, args)

    # If output is large, offload to file and return reference
    if estimate_tokens(raw_output) > 500:
        return fs_ctx.offload_output(f"{tool_name}_{hash(str(args))}", raw_output)

    return raw_output  # Small outputs stay in context
```

## Repository Structure

```
Agent-Skills-for-Context-Engineering/
├── .plugin/
│   └── plugin.json              # Open Plugins manifest
├── skills/
│   ├── context-fundamentals/    # What context is, anatomy, budgeting
│   ├── context-degradation/     # Failure patterns and diagnosis
│   ├── context-compression/     # Compression strategies
│   ├── context-optimization/    # KV-cache, masking, compaction
│   ├── multi-agent-patterns/    # Orchestrator, peer-to-peer, hierarchical
│   ├── memory-systems/          # Short/long-term, graph memory
│   ├── tool-design/             # MCP tools, agent-effective APIs
│   ├── filesystem-context/      # File-based context management
│   ├── hosted-agents/           # Sandboxed VMs, multiplayer agents
│   ├── evaluation/              # Test frameworks, metrics
│   ├── advanced-evaluation/     # LLM-as-Judge, rubrics, bias mitigation
│   ├── project-development/     # LLM project methodology
│   └── bdi-mental-states/       # BDI cognitive architecture
└── examples/
    ├── digital-brain-skill/     # Personal OS for founders/creators
    ├── x-to-book-system/        # Multi-agent X → book pipeline
    ├── llm-as-judge-skills/     # TypeScript evaluation tools (19 tests)
    └── book-sft-pipeline/       # Style transfer fine-tuning ($2 cost)
```

## Troubleshooting

### Skills not triggering automatically

```
Problem: Installed skills don't activate for relevant tasks.
Fix: Use explicit trigger phrases from the Skill Triggers table.
     Example: say "I want to implement agent memory" not just "add memory"
```

### Plugin install fails in Claude Code

```bash
# Verify marketplace is registered first
/plugin list marketplaces

# If not listed, re-add:
/plugin marketplace add muratcankoylan/Agent-Skills-for-Context-Engineering

# Then install:
/plugin install context-engineering-fundamentals@context-engineering-marketplace
```

### Context still degrading after applying skills

```python
# Checklist:
# 1. Are you placing important content in the middle? → Use arrange_context_for_attention()
# 2. Is tool output bloating context? → Use offload_output() for >500 token results
# 3. Is history unbounded? → Apply compress_conversation() after every N turns
# 4. Are you loading all tools always? → Filter to task-relevant tools only

def diagnose_context(messages: list[dict]) -> dict:
    total = sum(estimate_tokens(str(m)) for m in messages)
    return {
        "total_tokens": total,
        "over_budget": total > MAX_CONTEXT_TOKENS,
        "message_count": len(messages),
        "recommendation": "compress" if total > 6000 else "ok"
    }
```

### LLM-as-Judge giving inconsistent scores

```python
# Apply position bias mitigation: run judge twice with A/B swapped
# Use pairwise_compare() which handles this automatically.
# For direct scoring: run 3 times and average scores.

def robust_score(prompt, response, criteria, runs=3):
    scores = [direct_score(prompt, response, criteria).score for _ in range(runs)]
    return sum(scores) / len(scores)
```

### Memory retrieval returning irrelevant results

```python
# Current implementation uses keyword overlap (demonstration only).
# For production, replace with semantic search:

# Option 1: OpenAI embeddings
# Option 2: Local sentence-transformers
# Option 3: ChromaDB / Pinecone / Weaviate

# The MemoryEntry schema is compatible with any vector store —
# store entry.content as the document, asdict(entry) as metadata.
```

## Key Design Principles

1. **Smallest sufficient context**: Never include information not relevant to the current task
2. **Progressive disclosure**: Load skill details only when that skill is activated
3. **Append-only memory**: Never mutate history — append corrections and updates
4. **Attention-aware ordering**: High-value content at start and end of context
5. **Budget enforcement**: Assign explicit token budgets to each context segment
6. **File offloading**: Large tool outputs go to files; references stay in context
7. **Platform agnosticism**: Patterns work across Claude Code, Cursor, Codex, custom frameworks
```
