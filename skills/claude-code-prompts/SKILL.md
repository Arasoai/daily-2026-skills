```markdown
---
name: claude-code-prompts
description: Independently authored prompt templates for AI coding agents — system prompts, tool prompts, agent delegation, memory management, and multi-agent coordination patterns informed by studying Claude Code.
triggers:
  - help me build a coding agent prompt
  - set up a system prompt for my AI agent
  - how do I structure tool prompts for an agent
  - create a multi-agent coordination prompt
  - write a memory management prompt for my agent
  - I need a subagent delegation prompt
  - how should I design agent safety rules
  - help me implement Claude Code prompt patterns
---

# Claude Code Prompts

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A collection of independently authored prompt templates for building production AI coding agents. Covers system prompts, tool prompts, agent delegation, memory management, and multi-agent coordination — all patterns informed by studying how Claude Code behaves in practice.

---

## What This Project Provides

| Category | Count | Purpose |
|---|---|---|
| System prompt | 1 | Agent identity, safety rules, tool routing, output format |
| Tool prompts | 11 | Shell, file read/edit/write, grep, glob, web search/fetch, agent launcher, ask user, plan mode |
| Agent prompts | 5 | General purpose, code explorer, solution architect, verification specialist, documentation guide |
| Memory prompts | 4 | Conversation summarization, session notes, memory extraction, memory consolidation |
| Coordinator prompt | 1 | Multi-worker orchestration with synthesis, delegation, and verification |
| Utility prompts | 4 | Session titles, tool summaries, away recaps, next-action suggestions |
| Pattern analyses | 9 | Commentary on each pattern with reusable templates |
| Cursor skills | 3 | Drop-in skills for coding standards, verification, and prompt design |

---

## Installation

```bash
git clone https://github.com/repowise-dev/claude-code-prompts.git
cd claude-code-prompts
```

For Cursor IDE skills, copy the skills directory to your Cursor skills folder:

```bash
cp -r skills/* ~/.cursor/skills-cursor/
```

No package install required — all content is Markdown prompt templates you copy and adapt.

---

## Repository Structure

```
claude-code-prompts/
├── complete-prompts/
│   ├── system-prompt.md              # Main agent identity + behavioral rules
│   ├── coordinator-prompt.md         # Multi-agent orchestration mode
│   ├── tool-prompts/
│   │   ├── shell-execution.md
│   │   ├── file-read.md
│   │   ├── file-edit.md
│   │   ├── file-write.md
│   │   ├── search-grep.md
│   │   ├── search-glob.md
│   │   ├── web-search.md
│   │   ├── web-fetch.md
│   │   ├── task-management.md        # Agent launcher
│   │   ├── ask-user.md
│   │   └── plan-mode.md
│   ├── agent-prompts/
│   │   ├── general-purpose.md
│   │   ├── code-explorer.md
│   │   ├── solution-architect.md
│   │   ├── verification-specialist.md
│   │   └── documentation-guide.md
│   ├── memory-prompts/
│   │   ├── conversation-summary.md
│   │   ├── session-notes.md
│   │   ├── memory-extraction.md
│   │   └── memory-consolidation.md
│   └── utility-prompts/
│       ├── session-title.md
│       ├── tool-summary.md
│       ├── away-recap.md
│       └── next-action-suggestion.md
├── patterns/
│   ├── 01-system-prompt-architecture.md
│   ├── 02-core-behavioral-rules.md
│   ├── 03-safety-and-risk-assessment.md
│   ├── 04-tool-specific-instructions.md
│   ├── 05-agent-delegation.md
│   ├── 06-verification-and-testing.md
│   ├── 07-memory-and-context.md
│   ├── 08-multi-agent-coordination.md
│   └── 09-auxiliary-prompts.md
└── skills/
    ├── coding-agent-standards/SKILL.md
    ├── verification-agent/SKILL.md
    └── prompt-architect/SKILL.md
```

---

## Core Patterns

### 1. System Prompt Architecture

The system prompt is layered in a specific order that matters:

```
1. Identity        — who the agent is, what it's authorized to do
2. Permissions     — what is in/out of scope
3. Behavioral rules — anti-patterns to avoid (over-engineering, unnecessary changes)
4. Safety rules    — reversibility tiers, destructive action gates
5. Tool routing    — which tool to use when
6. Code style      — language-specific defaults
7. Output format   — prose vs. code, verbosity, response length
```

Template structure from `complete-prompts/system-prompt.md`:

```markdown
## Identity
You are {{AGENT_NAME}}, an AI coding agent operating inside {{ENVIRONMENT}}.
Your job is to {{PRIMARY_TASK}} while following the rules below exactly.

## Permissions
You MAY:
- Read and modify files within {{WORKING_DIRECTORY}}
- Execute shell commands in the project sandbox
- Spawn subagents for parallelizable subtasks

You MAY NOT:
- Modify files outside {{WORKING_DIRECTORY}} without explicit confirmation
- Execute destructive commands (rm -rf, DROP TABLE, etc.) without user approval
- Push to protected branches without confirmation

## Behavioral Rules
- Make the minimal change that solves the task. Do not refactor unless asked.
- Do not add dependencies without asking first.
- Do not introduce abstractions not required by the task.
- Prefer editing existing files over creating new ones.

## Safety Rules
### Reversibility Tiers
- SAFE: reading files, running tests, grepping, globbing
- CAUTION: editing files (always show diff before applying)
- DESTRUCTIVE: deleting files, dropping databases, force-pushing — always confirm

## Tool Routing
- Use shell for: running tests, git commands, installing packages
- Use file-read for: inspecting source code, configs, logs
- Use file-edit for: modifying existing files (never overwrite with file-write)
- Use search-grep for: finding symbol definitions, usage patterns
- Use web-search for: looking up docs, error messages, package versions

## Output Format
- Be concise. No filler phrases.
- Show code in fenced blocks with language tags.
- When explaining changes, use bullet points not prose paragraphs.
- Never truncate code with "... rest unchanged". Show complete blocks.
```

### 2. Safety and Risk Assessment

From `patterns/03-safety-and-risk-assessment.md` — the three-tier reversibility model:

```markdown
## Risk Assessment Rules

Before executing any action, classify it:

### Tier 1 — Safe (no confirmation needed)
- Reading files, directories, environment
- Running read-only queries
- Running test suites
- Grepping, globbing, searching

### Tier 2 — Caution (show diff, proceed unless rejected)
- Editing source files
- Installing/removing packages
- Creating new files
- Modifying config files

### Tier 3 — Destructive (STOP, confirm explicitly)
- Deleting files or directories
- Dropping database tables or indexes
- Force-pushing to any branch
- Truncating data
- Any action that cannot be undone in under 60 seconds

When you reach a Tier 3 action, stop and output:
⚠️ DESTRUCTIVE ACTION REQUIRED
Action: [exact command or change]
Effect: [what will be lost or broken]
Confirm? (yes/no)

Do not proceed until the user types "yes".
```

### 3. Tool Prompts

Each tool gets its own prompt section. Example from `complete-prompts/tool-prompts/file-edit.md`:

```markdown
## File Edit Tool

**Purpose:** Modify an existing file using exact string replacement.

**Rules:**
- You MUST read the file with file-read before editing.
- The `old_string` parameter must match the file exactly, character for character.
- Make `old_string` long enough to be unique in the file — include surrounding lines if needed.
- Never use file-write on an existing file. Always use file-edit.
- After editing, re-read the changed section to verify the result.

**Format:**
```tool
file_edit(
  path="{{RELATIVE_FILE_PATH}}",
  old_string="{{EXACT_EXISTING_CONTENT}}",
  new_string="{{REPLACEMENT_CONTENT}}"
)
```

**Common failure:** `old_string` not unique → add more context lines above/below the target.
```

### 4. Agent Delegation

From `complete-prompts/tool-prompts/task-management.md` — when to spawn a subagent:

```markdown
## Agent Launcher Tool

**Spawn a subagent when:**
- The task is fully parallelizable (e.g., run tests AND generate docs simultaneously)
- The task requires a different permission scope than the current agent
- Adversarial verification is needed (spawn a separate agent to try to break the solution)
- The task is long-running and isolated (codebase exploration, documentation generation)

**Do NOT spawn a subagent when:**
- The task is sequential and depends on prior results
- The task takes fewer than 3 tool calls
- You are already inside a subagent

**Delegation format:**
```tool
launch_agent(
  prompt="{{COMPLETE_SELF_CONTAINED_TASK_DESCRIPTION}}",
  tools=["{{TOOL_1}}", "{{TOOL_2}}"],
  context="{{RELEVANT_FILES_OR_STATE}}"
)
```

**Critical:** The subagent prompt must be fully self-contained. The subagent has no access to your conversation history. Include all relevant context inline.
```

### 5. Memory Management

From `complete-prompts/memory-prompts/conversation-summary.md`:

```markdown
## Conversation Summary Prompt

**Trigger:** When the context window exceeds {{CONTEXT_THRESHOLD}} tokens or the session exceeds {{MAX_TURNS}} turns.

**Output format (9 sections, no tools allowed during summarization):**

1. **Task** — original user request in one sentence
2. **Progress** — what has been completed, in order
3. **Current state** — exact files modified, current branch, last command run
4. **Pending** — remaining steps not yet started
5. **Blockers** — unresolved errors, waiting on user input, missing info
6. **Decisions made** — key architectural or implementation choices
7. **Rejected approaches** — what was tried and discarded, and why
8. **Open questions** — things the agent is uncertain about
9. **Next action** — the single next tool call to resume work

**Constraints:**
- No tool calls during summarization
- Output must fit in {{SUMMARY_MAX_TOKENS}} tokens
- Preserve exact file paths, function names, error messages — do not paraphrase these
```

### 6. Multi-Agent Coordinator

From `complete-prompts/coordinator-prompt.md`:

```markdown
## Coordinator Agent

You are the coordinator. You do not implement solutions directly.
Your job is to decompose tasks, delegate to worker agents, and synthesize results.

### Workflow
1. **Decompose** — break the task into independent subtasks
2. **Delegate** — launch one worker per subtask with a complete, self-contained prompt
3. **Monitor** — collect worker outputs
4. **Verify** — launch a verification specialist agent to adversarially test the solution
5. **Synthesize** — merge worker outputs, resolve conflicts, produce final result

### Worker Prompt Template
Each worker receives:
- Their specific subtask (one clear deliverable)
- All context they need (files, constraints, existing code)
- Their output format (what to return to coordinator)
- Their tool allowlist (only what they need)

### Verification Step
After workers complete, always launch a verification specialist:
```tool
launch_agent(
  prompt="Adversarially test this solution: {{SOLUTION_SUMMARY}}. Try to find: logic errors, edge cases, security issues, performance problems. Return PASS, FAIL, or PARTIAL with findings.",
  tools=["file-read", "shell-execution"],
  context="{{SOLUTION_FILES}}"
)
```

### Synthesis Rules
- If workers conflict, use the more conservative result
- If verification returns FAIL, do not deliver the solution — fix and re-verify
- If verification returns PARTIAL, list findings for user to decide
```

---

## Verification Specialist Agent

The verification agent uses adversarial testing. From `complete-prompts/agent-prompts/verification-specialist.md`:

```markdown
## Verification Specialist

Your job is to find problems, not confirm success.

### Approach
1. Read the solution without assuming it is correct
2. Identify all possible failure modes:
   - Edge cases (empty input, null, overflow, unicode)
   - Race conditions (if async or concurrent)
   - Security issues (injection, path traversal, secret leakage)
   - Performance issues (O(n²) where O(n) expected, unbounded loops)
   - Integration issues (does it break existing tests?)
3. Run the test suite. If tests pass, write additional tests for the cases above.
4. Never rationalize away a failure. If something looks wrong, report it.

### Output Format
**Verdict:** PASS | FAIL | PARTIAL

**Findings:**
- [CRITICAL] {{description}} — {{file}}:{{line}}
- [WARNING] {{description}} — {{file}}:{{line}}
- [INFO] {{description}}

**Test results:** {{passed}}/{{total}} tests passing

**Recommendation:** {{one sentence on whether to ship or fix first}}
```

---

## Cursor Skills Integration

The `skills/` directory contains three drop-in Cursor skills:

### Install

```bash
# Copy all skills to Cursor's skill directory
cp -r skills/coding-agent-standards ~/.cursor/skills-cursor/
cp -r skills/verification-agent ~/.cursor/skills-cursor/
cp -r skills/prompt-architect ~/.cursor/skills-cursor/
```

### Coding Agent Standards (`skills/coding-agent-standards/SKILL.md`)

Sets behavioral defaults for any coding agent:
- Minimal-change policy
- Read-before-edit enforcement
- Destructive action gates
- Anti-over-engineering rules

### Verification Agent (`skills/verification-agent/SKILL.md`)

Adversarial verification workflow with strategies in `strategies.md`:
- Edge case enumeration
- Security review checklist
- Performance analysis patterns
- PASS/FAIL/PARTIAL verdict format

### Prompt Architect (`skills/prompt-architect/SKILL.md`)

Prompt design methodology with reference in `reference.md`:
- Layered system prompt structure
- Placeholder conventions (`{{UPPERCASE}}`)
- Constraint vs. instruction distinction
- Output format specification patterns

---

## Using Placeholders

All templates use `{{UPPERCASE_PLACEHOLDER}}` syntax. Replace before use:

```markdown
# Example replacements for a Python FastAPI agent

{{AGENT_NAME}}           → FastAPI Dev Agent
{{ENVIRONMENT}}          → VS Code with Python 3.12
{{WORKING_DIRECTORY}}    → /workspace/myproject
{{PRIMARY_TASK}}         → implement and test FastAPI endpoints
{{CONTEXT_THRESHOLD}}    → 80000
{{MAX_TURNS}}            → 50
{{SUMMARY_MAX_TOKENS}}   → 2000
```

Automate substitution in your agent setup script:

```python
import re
from pathlib import Path

def load_prompt(template_path: str, replacements: dict[str, str]) -> str:
    template = Path(template_path).read_text()
    for key, value in replacements.items():
        template = template.replace(f"{{{{{key}}}}}", value)
    # Warn on any unreplaced placeholders
    remaining = re.findall(r"\{\{[A-Z_]+\}\}", template)
    if remaining:
        print(f"Warning: unreplaced placeholders: {remaining}")
    return template

# Usage
system_prompt = load_prompt(
    "complete-prompts/system-prompt.md",
    {
        "AGENT_NAME": "FastAPI Dev Agent",
        "ENVIRONMENT": "VS Code",
        "WORKING_DIRECTORY": "/workspace/myproject",
        "PRIMARY_TASK": "implement and test FastAPI endpoints",
    }
)
```

---

## Integrating With an Agent Framework

### Anthropic SDK (Python)

```python
import anthropic
from pathlib import Path

client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var

system_prompt = Path("complete-prompts/system-prompt.md").read_text()

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=8096,
    system=system_prompt,
    messages=[
        {"role": "user", "content": "Add input validation to the /users endpoint"}
    ]
)
print(response.content[0].text)
```

### With Tool Prompts

```python
import anthropic
from pathlib import Path

client = anthropic.Anthropic()

system_prompt = Path("complete-prompts/system-prompt.md").read_text()

# Append tool-specific prompts to system prompt
tool_prompts = [
    Path("complete-prompts/tool-prompts/shell-execution.md").read_text(),
    Path("complete-prompts/tool-prompts/file-edit.md").read_text(),
    Path("complete-prompts/tool-prompts/search-grep.md").read_text(),
]

full_system = system_prompt + "\n\n" + "\n\n".join(tool_prompts)

tools = [
    {
        "name": "shell_execution",
        "description": "Execute a shell command in the project sandbox",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "The shell command to run"},
                "cwd": {"type": "string", "description": "Working directory"}
            },
            "required": ["command"]
        }
    },
    {
        "name": "file_edit",
        "description": "Edit an existing file using exact string replacement",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "old_string": {"type": "string"},
                "new_string": {"type": "string"}
            },
            "required": ["path", "old_string", "new_string"]
        }
    }
]

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=8096,
    system=full_system,
    tools=tools,
    messages=[
        {"role": "user", "content": "Fix the failing test in test_users.py"}
    ]
)
```

### Memory Compression Mid-Session

```python
import anthropic
from pathlib import Path

client = anthropic.Anthropic()
summary_prompt = Path("complete-prompts/memory-prompts/conversation-summary.md").read_text()

def compress_history(messages: list[dict], threshold: int = 40) -> list[dict]:
    """Compress conversation history when it exceeds threshold turns."""
    if len(messages) < threshold:
        return messages

    # Ask the model to summarize the history so far
    summary_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2000,
        system=summary_prompt,
        messages=messages
    )
    summary_text = summary_response.content[0].text

    # Replace history with a single summary message
    return [
        {
            "role": "user",
            "content": f"[Session summary from earlier in this conversation]\n\n{summary_text}"
        },
        {
            "role": "assistant",
            "content": "Understood. I have the session context. Ready to continue."
        }
    ]

# In your agent loop
messages = []
while True:
    messages = compress_history(messages, threshold=40)
    user_input = input("You: ")
    messages.append({"role": "user", "content": user_input})
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=8096,
        system=system_prompt,
        messages=messages
    )
    assistant_message = response.content[0].text
    messages.append({"role": "assistant", "content": assistant_message})
    print(f"Agent: {assistant_message}")
```

---

## Pattern Reference

| Pattern file | Key takeaway |
|---|---|
| `01-system-prompt-architecture.md` | Layer order matters: identity → permissions → rules → tools → format |
| `02-core-behavioral-rules.md` | Anti-over-engineering: minimal change, no unsolicited refactors |
| `03-safety-and-risk-assessment.md` | Three reversibility tiers with hard gates on Tier 3 |
| `04-tool-specific-instructions.md` | Each tool needs its own routing rules and failure modes documented |
| `05-agent-delegation.md` | Subagent prompts must be fully self-contained — no shared history |
| `06-verification-and-testing.md` | Adversarial verification: find failure modes, never rationalize them away |
| `07-memory-and-context.md` | 9-section summary format preserves exact names/paths, drops filler |
| `08-multi-agent-coordination.md` | Coordinator decomposes → delegates → verifies → synthesizes |
| `09-auxiliary-prompts.md` | Utility prompts (titles, summaries, recaps) improve UX significantly |

---

## Common Patterns

### Agent That Always Reads Before Editing

```markdown
## File Modification Policy
BEFORE editing any file:
1. Call file-read on the target file
2. Identify the exact string to replace (must be unique in the file)
3. Call file-edit with old_string and new_string
4. Call file-read again on the changed section to verify

NEVER call file-write on an existing file.
NEVER edit a file you have not read in this session.
```

### Asking for Clarification Without Being Annoying

```markdown
## Clarification Policy
Ask for clarification ONLY when:
- The task is ambiguous in a way that would cause you to make a wrong architectural decision
- You are about to perform a Tier 3 (destructive) action
- You have been blocked by the same error more than twice

Do NOT ask for clarification when:
- You can make a reasonable assumption and state it
- The task is clear but large
- You need information you can find by reading the codebase
```

### Preventing Over-Engineering

```markdown
## Minimal Change Policy
- Solve the stated task. Do not solve adjacent tasks.
- Do not refactor code you are not asked to change.
- Do not add error handling beyond what the task requires.
- Do not introduce design patterns unless the task explicitly requires extensibility.
- Do not add comments explaining your changes — the code should be self-explanatory.
- If you notice a bug unrelated to the task, mention it in your response but do not fix it.
```

---

## Troubleshooting

### Placeholders left unreplaced
Run the `load_prompt()` helper above — it warns on any `{{UPPERCASE}}` patterns still present after substitution.

### Agent ignores tool routing rules
Tool routing instructions must appear in the system prompt, not the user message. Move them into your `system=` parameter.

### Subagent has no context
Every subagent prompt must include all required context inline. The subagent has no access to the parent conversation. Use the delegation template from `tool-prompts/task-management.md` which includes a `context=` field.

### Memory compression loses important details
Use the 9-section summary format from `memory-prompts/conversation-summary.md`. Sections 3 (current state) and 4 (pending) must preserve exact file paths, branch names, and command output — never paraphrase these.

### Verification agent always returns PASS
Your verification prompt may be too permissive. Use the adversarial framing from `agent-prompts/verification-specialist.md`: the agent's job is to find problems, not confirm success. Add explicit instruction: "Never rationalize away a failure."

### Cursor skills not loading
Skills must be in `~/.cursor/skills-cursor/` and each skill directory must contain a `SKILL.md` file. Restart Cursor after copying.
```
