```markdown
---
name: claude-code-system-prompts-research
description: Research documentation of Claude Code's internal prompt architecture, agent directives, multi-agent orchestration patterns, and security classifiers
triggers:
  - "how does claude code's system prompt work"
  - "explain claude code agent architecture"
  - "show me the auto mode classifier design"
  - "how does claude code handle multi-agent orchestration"
  - "what are claude code's security boundaries"
  - "how does prompt caching work in claude code"
  - "explain claude code memory system"
  - "how does context window management work in claude code"
---

# Claude Code System Prompts Research

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

An independent research project documenting the internal prompt architecture, agent directives, and security classifiers of **Claude Code** — Anthropic's AI-powered software engineering assistant. Use this skill to understand design patterns for production-grade agentic AI systems.

---

## What This Project Documents

This repository catalogs 30+ prompts recovered through behavioral analysis and output observation. It covers:

- **Core identity prompts** — how Claude Code defines itself and its constraints
- **Multi-agent orchestration** — coordinator patterns, sub-agent spawning, swarm communication
- **Security classifiers** — 2-stage auto-approval pipeline for tool calls
- **Context window management** — compaction, caching, micro-summarization
- **Memory systems** — hierarchical CLAUDE.md loading with override semantics
- **Specialized agents** — verification, exploration, agent-creation, browser automation

---

## Repository Structure

```
claude-code-system-prompts/
    README.md
    prompts/
        01_main_system_prompt.md         # Master assembled prompt
        02_simple_mode.md                # CLAUDE_CODE_SIMPLE minimal prompt
        03_default_agent_prompt.md       # Base inherited by all sub-agents
        04_cyber_risk_instruction.md     # Security allow/deny boundaries
        05_coordinator_system_prompt.md  # Multi-worker orchestrator
        06_teammate_prompt_addendum.md   # Swarm/team communication protocol
        07_verification_agent.md         # Adversarial testing specialist
        08_explore_agent.md              # Read-only codebase explorer
        09_agent_creation_architect.md   # Designs new agent configs
        10_statusline_setup_agent.md     # Terminal status line setup
        11_permission_explainer.md       # Tool risk explanations
        12_yolo_auto_mode_classifier.md  # 2-stage security classifier
        13_tool_prompts.md               # All 30+ tool descriptions
        14_tool_use_summary.md           # Git-commit-style tool labels
        15_session_search.md             # Semantic session search
        16_memory_selection.md           # Memory file relevance selector
        17_auto_mode_critique.md         # Reviews classifier rules
        18_proactive_mode.md             # Autonomous tick-based agent
        19_simplify_skill.md             # 3-agent parallel code review
        20_session_title.md              # Session title generator
        21_compact_service.md            # Conversation summarization
        22_away_summary.md               # Idle session recap
        23_chrome_browser_automation.md  # Browser extension integration
        24_memory_instruction.md         # CLAUDE.md @include directives
        25_skillify.md                   # Skill creation interview
        26_stuck_skill.md                # Frozen session diagnostics
        27_remember_skill.md             # Promotes memory to CLAUDE.md
        28_update_config_skill.md        # settings.json/hooks manager
        29_agent_summary.md              # Sub-agent progress updates
        30_prompt_suggestion.md          # Predicts next user command
```

---

## Key Architecture Patterns

### 1. Dynamic Prompt Assembly

The main system prompt is not a static string. It is assembled at runtime from modular section-builder functions with a cache boundary splitting static and dynamic content:

```
┌─────────────────────────────────────────┐
│         STATIC PREFIX (cached globally) │
│  - Identity + Cyber Risk                │
│  - Permission modes, hooks, reminders   │
│  - Code style, security, reversibility  │
│  - Tool preferences, parallel calls     │
│  - Tone, output efficiency patterns     │
├─────────────────────────────────────────┤
│              CACHE BOUNDARY             │
├─────────────────────────────────────────┤
│      DYNAMIC SUFFIX (session-specific)  │
│  - Agent tools, skills, verification    │
│  - Memory file content (CLAUDE.md)      │
│  - Model overrides                      │
│  - Environment: CWD, OS, git state      │
│  - Language preferences                 │
│  - MCP server instructions              │
│  - Context window state                 │
└─────────────────────────────────────────┘
```

**Key insight for your own agents:** Separate stable behavioral instructions (cache-eligible) from runtime context (session-specific). This reduces token costs significantly on repeated calls.

### 2. 2-Stage Auto-Mode Security Classifier

The `12_yolo_auto_mode_classifier.md` describes a pipeline for safely auto-approving tool calls:

```
Tool Call Request
       │
       ▼
┌─────────────┐
│   Stage 1   │  Fast classification
│  Classifier │  Allow / Deny / Uncertain
└─────────────┘
       │
  Uncertain?
       │
       ▼
┌─────────────┐
│   Stage 2   │  Extended thinking
│  Classifier │  Allow / Deny
└─────────────┘
```

Classifier prompt is assembled from four layers:

```
Base classifier instructions
       +
Default rules (allow / deny / environment sections)
       +
User overrides (replace entire sections)
       +
Runtime context (tool name, args, working directory)
```

**Pattern to replicate:**

```python
def build_classifier_prompt(
    base: str,
    default_rules: dict,
    user_overrides: dict,
    tool_call: dict
) -> str:
    rules = {**default_rules, **user_overrides}  # overrides win
    return f"""
{base}

## Rules
### Allow
{rules.get('allow', '')}

### Deny
{rules.get('deny', '')}

### Environment
{rules.get('environment', '')}

## Tool Call to Classify
Tool: {tool_call['name']}
Arguments: {tool_call['args']}
Working Directory: {tool_call['cwd']}
"""
```

### 3. Multi-Agent Coordinator (4-Phase Workflow)

From `05_coordinator_system_prompt.md`, the orchestrator follows a structured loop:

```
Phase 1: PLAN
  - Decompose task into parallel work units
  - Identify dependencies between units
  - Determine concurrency limits

Phase 2: SPAWN
  - Launch worker agents with isolated contexts
  - Assign each a specific sub-task and tool subset
  - Set verification requirements

Phase 3: MONITOR
  - Collect periodic progress summaries (prompt 29)
  - Detect blocked or stuck agents (prompt 26)
  - Re-assign failed work units

Phase 4: INTEGRATE
  - Merge outputs from all workers
  - Run verification agent (prompt 07) against result
  - Report to user with summary
```

**Spawning a sub-agent (conceptual pattern):**

```python
def spawn_worker(task: str, tools: list[str], read_only: bool = False) -> Agent:
    base = load_prompt("03_default_agent_prompt.md")
    if read_only:
        base += load_prompt("08_explore_agent.md")
    return Agent(
        system_prompt=base,
        allowed_tools=tools,
        task=task,
        reports_to="coordinator"
    )

# Parallel exploration example
workers = [
    spawn_worker("Map all API endpoints", tools=["read_file", "grep"], read_only=True),
    spawn_worker("Find all database models", tools=["read_file", "grep"], read_only=True),
    spawn_worker("List all test files", tools=["read_file", "glob"], read_only=True),
]
results = await asyncio.gather(*[w.run() for w in workers])
```

### 4. Memory System (Hierarchical CLAUDE.md)

From `24_memory_instruction.md`, memory files are loaded in priority order:

```
Priority (lowest → highest):
  1. Enterprise managed config       (~/.claude/enterprise/CLAUDE.md)
  2. User global config              (~/.claude/CLAUDE.md)
  3. Project shared config           ./CLAUDE.md
  4. Project rules directory         ./CLAUDE/
  5. Local private config            ./.claude/CLAUDE.md  (git-ignored)
```

**@include directive syntax (resolved transitively, max depth 5):**

```markdown
<!-- In CLAUDE.md -->
@include ./CLAUDE/backend-rules.md
@include ./CLAUDE/testing-standards.md
```

**Frontmatter glob injection (only inject for matching files):**

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/routes/**/*.ts"
---
# API Layer Rules
- Always validate request bodies with zod
- Return RFC 7807 problem+json for errors
```

**Memory selection prompt (16) picks relevant files:**

```python
def select_memory_files(query: str, available_files: list[str]) -> list[str]:
    """
    Uses prompt 16 to semantically rank memory files by relevance to
    the current query, then injects only the top-K into context.
    """
    prompt = load_prompt("16_memory_selection.md")
    response = claude(prompt + f"\nQuery: {query}\nFiles:\n" + "\n".join(available_files))
    return parse_selected_files(response)
```

### 5. Context Window Pipeline

```
New User Message
       │
       ▼
[Micro-Compaction]     Delete stale tool results (cache-aware)
       │
       ▼
[Compact Service]      Full or partial summarization
       │   (21_compact_service.md — analysis + summary blocks)
       ▼
[Prompt Suggestion]    Predict next command (30_prompt_suggestion.md)
       │
       ▼
[Away Summary]         1-3 sentence recap if user was idle
                       (22_away_summary.md)
```

**Compact service variants (from prompt 21):**

| Mode | Behavior |
|------|----------|
| Full compaction | Entire conversation → single summary block |
| Partial compaction | Keep recent N turns, summarize the rest |
| Analysis block | Structured reasoning about what happened |
| Summary block | Human-readable recap injected as assistant turn |

---

## Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_SIMPLE=1` | Activates minimal 4-line system prompt (prompt 02) |
| `USER_TYPE=ant` | Enables internal-only Anthropic sections and model overrides |
| Feature flags (various) | Gate proactive mode, fork subagents, verification agents |

---

## Specialized Agent Patterns

### Verification Agent (Adversarial Tester)

From `07_verification_agent.md` — spawned after implementation to attempt to break it:

```python
verification_prompt = """
You are an adversarial testing specialist. Your job is to BREAK the
implementation you are given. Try:
- Edge cases the implementer didn't consider
- Boundary conditions (empty, null, max values)
- Concurrent access patterns
- Invalid inputs that should be rejected
- Performance degradation scenarios

Report every failure. Do not stop at the first bug.
"""
```

### Explore Agent (Read-Only Constraint)

From `08_explore_agent.md` — strict no-modify enforced at prompt level:

```python
explore_prompt = """
You have READ-ONLY access to this codebase. You MUST NOT:
- Write, create, or delete any files
- Execute any commands that modify state
- Make any network requests

Your only job is to understand and report.
"""
```

### Proactive Mode (Tick-Based Autonomous Agent)

From `18_proactive_mode.md`:

```
Every tick:
  1. Check if terminal has focus (pause if backgrounded)
  2. Assess current task state
  3. Take highest-priority autonomous action
  4. Report progress
  5. Check for user interruption signal
  6. Sleep until next tick
```

---

## Skills System (SKILL.md Pattern)

Claude Code includes a meta-skill system. From `25_skillify.md`:

```
User: "I want to create a skill for X"
       │
       ▼
Interview Phase:
  - What triggers should activate this skill?
  - What are the most common tasks?
  - What tools does the project provide?
  - What are the gotchas?
       │
       ▼
Generation Phase:
  - Produce SKILL.md with YAML frontmatter
  - Include triggers, examples, patterns
  - Save to .claude/skills/ or project root
```

Skills are loaded by the memory system when triggers match. The `27_remember_skill.md` prompt promotes ephemeral memories into persistent CLAUDE.md entries.

---

## Applying These Patterns

### Building Your Own Multi-Agent System

```python
import anthropic

client = anthropic.Anthropic()

COORDINATOR_PROMPT = """
You are a multi-agent coordinator. Given a task:
1. Decompose into parallel subtasks
2. Specify which tools each worker needs
3. Identify which subtasks block others
4. Return a JSON work plan
"""

WORKER_BASE_PROMPT = """
You are a specialized worker agent. Complete your assigned subtask.
Report your findings clearly. Do not modify files unless explicitly authorized.
"""

def run_coordinator(task: str) -> dict:
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        system=COORDINATOR_PROMPT,
        messages=[{"role": "user", "content": task}]
    )
    return parse_work_plan(response.content[0].text)

def run_worker(subtask: str, tools: list) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5",  # cheaper for workers
        max_tokens=4096,
        system=WORKER_BASE_PROMPT,
        messages=[{"role": "user", "content": subtask}],
        tools=tools
    )
    return response.content[0].text
```

### Implementing a Simple 2-Stage Classifier

```python
def classify_tool_call(tool_name: str, tool_args: dict, rules: dict) -> str:
    """Returns 'allow', 'deny', or routes to stage 2 for 'uncertain'."""

    # Stage 1: fast pattern matching
    for pattern in rules.get("deny", []):
        if matches(tool_name, tool_args, pattern):
            return "deny"
    for pattern in rules.get("allow", []):
        if matches(tool_name, tool_args, pattern):
            return "allow"

    # Stage 2: extended thinking for uncertain cases
    prompt = build_classifier_prompt(tool_name, tool_args, rules)
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=8000,
        thinking={"type": "enabled", "budget_tokens": 5000},
        system="You are a security classifier. Respond with only: allow or deny.",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[-1].text.strip().lower()
```

---

## Troubleshooting / Research Gaps

| Issue | Notes |
|-------|-------|
| Prompt numbers are non-contiguous | Gaps (e.g., no 00, gap after 18) suggest internal versioning or removed prompts |
| Feature flag names not documented | Flags gating proactive mode and fork subagents are referenced but not enumerated |
| Exact cache boundary marker unknown | Architecture is inferred; actual token or string boundary is not confirmed |
| `USER_TYPE=ant` behavior | Internal sections are described but not fully reproduced |
| Tool descriptions may drift | Prompt 13 covers 30+ tools; Claude Code updates may add/remove tools |

---

## References

- [`prompts/01_main_system_prompt.md`](prompts/01_main_system_prompt.md) — Start here for core identity
- [`prompts/05_coordinator_system_prompt.md`](prompts/05_coordinator_system_prompt.md) — Multi-agent patterns
- [`prompts/12_yolo_auto_mode_classifier.md`](prompts/12_yolo_auto_mode_classifier.md) — Security classifier
- [`prompts/21_compact_service.md`](prompts/21_compact_service.md) — Context window management
- [`prompts/24_memory_instruction.md`](prompts/24_memory_instruction.md) — Memory/CLAUDE.md system
```
