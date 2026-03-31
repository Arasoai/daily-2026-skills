```markdown
---
name: claude-code-deep-dive
description: Expertise in the Claude Code deep dive research report — navigating the extracted source, understanding the agent architecture, prompt assembly system, tool execution pipeline, and all major subsystems reverse-engineered from the npm package source map.
triggers:
  - "explain how Claude Code works internally"
  - "show me the claude code architecture"
  - "how does claude code assemble its system prompt"
  - "explain the agent tool and subagent system"
  - "how do skills plugins and hooks work in claude code"
  - "walk me through the tool execution pipeline"
  - "what files are in the extracted source"
  - "how does claude code handle permissions and mcp"
---

# Claude Code Deep Dive

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

This repository is a **systematic reverse-engineering research report** of `@anthropic-ai/claude-code` — reconstructed from the `cli.js.map` source map bundled inside the published npm package. The map's `sourcesContent` field contained **4,756 original TypeScript source files**, making full architectural analysis possible.

The report covers:
- Complete source structure of Claude Code
- How the system prompt is dynamically assembled
- The AgentTool / SkillTool protocol
- Built-in agent roles and the agent dispatch chain
- Plugin / Skill / Hook / MCP runtime integration
- Permission and tool execution pipelines
- Why Claude Code behaves more like an Agent Operating System than a simple CLI wrapper

### Primary Artifacts

| Path | Description |
|---|---|
| `claude-code-deep-dive-xelatex.pdf` | Full research report (PDF) |
| `extracted-source/` | All 4,756 reconstructed TypeScript source files |
| `extracted-source/src/constants/prompts.ts` | Main system prompt assembly |
| `extracted-source/src/tools/AgentTool/` | Agent dispatch system |
| `extracted-source/src/tools/SkillTool/` | Skill invocation system |
| `extracted-source/src/services/tools/` | Tool execution + hook pipeline |

---

## Navigating the Extracted Source

```bash
# Clone the repo
git clone https://github.com/tvytlx/claude-code-deep-dive.git
cd claude-code-deep-dive

# Top-level source layout
ls extracted-source/src/
# entrypoints/   constants/   tools/   services/   utils/
# commands/      components/  coordinator/  memdir/
# plugins/       hooks/       bootstrap/    tasks/
```

### Key File Locations

```
extracted-source/src/
├── constants/
│   └── prompts.ts                    # ← THE main system prompt assembler
├── tools/
│   ├── AgentTool/
│   │   ├── AgentTool.tsx             # Agent tool definition + UI
│   │   ├── runAgent.ts               # Agent execution loop
│   │   └── prompt.ts                 # Agent-specific prompt fragments
│   ├── SkillTool/
│   │   ├── SkillTool.tsx
│   │   └── prompt.ts
│   ├── FileRead.ts
│   ├── FileEdit.ts
│   ├── FileWrite.ts
│   ├── Bash.ts
│   ├── Glob.ts
│   ├── Grep.ts
│   └── TodoWrite.ts
├── services/
│   ├── tools/
│   │   ├── toolExecution.ts          # Full tool execution pipeline
│   │   └── toolHooks.ts             # Pre/post hook integration
│   └── mcp/                         # MCP bridge
├── entrypoints/
│   ├── cli.tsx                       # Main CLI entry
│   ├── init.ts
│   ├── mcp.ts                        # MCP server mode
│   └── sdk/                          # SDK consumer entry
└── commands.ts                       # All slash commands registered
```

---

## Core Architecture Concepts

### 1. System Prompt Assembly (`prompts.ts`)

Claude Code does **not** use a static system prompt. `getSystemPrompt()` is a runtime assembler:

```typescript
// Conceptual reconstruction of getSystemPrompt()
function getSystemPrompt(context: SessionContext): string {
  // STATIC PREFIX — suitable for prompt caching
  const staticSections = [
    getSimpleIntroSection(),           // Identity + CYBER_RISK_INSTRUCTION
    getSimpleSystemSection(),          // Runtime rules (hooks, compression, tags)
    getSimpleDoingTasksSection(),      // Engineering behavior constraints
    getActionsSection(),               // Risk/destructive action rules
    getUsingYourToolsSection(),        // Tool grammar (FileRead > cat, etc.)
    getSimpleToneAndStyleSection(),
    getOutputEfficiencySection(),
  ].join('\n');

  // DYNAMIC SUFFIX — session-specific, NOT cached
  // Boundary: SYSTEM_PROMPT_DYNAMIC_BOUNDARY
  const dynamicSections = [
    getSessionSpecificGuidanceSection(context),  // Current tool set + feature gates
    getMemorySection(context),
    getEnvInfoSection(context),
    getLanguageSection(context),
    getOutputStyleSection(context),
    getMcpInstructionsSection(context),
    getScratchpadSection(context),
    getFunctionResultClearingSection(context),
    getTokenBudgetSection(context),
  ].filter(Boolean).join('\n');

  return staticSections + SYSTEM_PROMPT_DYNAMIC_BOUNDARY + dynamicSections;
}
```

**Key insight**: The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker separates cache-friendly static content from per-session dynamic content. Moving content across this boundary has cost implications.

### 2. Agent Dispatch Chain

```
User Request
     │
     ▼
AgentTool.tsx          ← tool definition, permission check, UI rendering
     │
     ▼
runAgent.ts            ← spawns isolated agent context
     │  - forks session state
     │  - injects agent-specific system prompt (prompt.ts)
     │  - sets up sub-tool allowlist
     ▼
query()                ← model inference loop (shared with main loop)
     │
     ▼
toolExecution.ts       ← same pipeline as main agent
```

```typescript
// Simplified agent invocation pattern (from AgentTool reconstruction)
const agentResult = await runAgent({
  task: toolInput.task,
  parentSession: currentSession,
  allowedTools: resolveAgentTools(context),   // Subset of parent tools
  agentPrompt: getAgentPrompt(agentType),     // From AgentTool/prompt.ts
  mcpTools: filteredMcpTools,
});
```

### 3. Tool Execution Pipeline

Every tool call (including MCP tools) passes through:

```
Model emits tool_use block
         │
         ▼
toolExecution.ts
  ├── 1. Permission check (permission mode + user grants)
  ├── 2. Pre-execution hooks (toolHooks.ts → external hook scripts)
  ├── 3. Actual tool.call() invocation
  ├── 4. Post-execution hooks
  ├── 5. Analytics event emission
  └── 6. Result returned as tool_result block
```

```typescript
// Conceptual pipeline (reconstructed from toolExecution.ts)
async function executeTool(tool: Tool, input: unknown, ctx: ExecContext) {
  // Step 1: Permission
  const permitted = await checkPermission(tool, input, ctx);
  if (!permitted) return permissionDeniedResult(tool);

  // Step 2: Pre-hook
  const hookDecision = await runPreHooks(tool, input, ctx);
  if (hookDecision.block) return hookDecision.result;

  // Step 3: Execute
  const result = await tool.call(input, ctx);

  // Step 4: Post-hook
  await runPostHooks(tool, input, result, ctx);

  // Step 5: Analytics
  emitToolEvent(tool.name, input, result);

  return result;
}
```

### 4. Built-in Agent Types

From `AgentTool/prompt.ts` and session guidance sections:

| Agent Type | Role |
|---|---|
| Default subagent | General task execution with full tool access |
| Explore agent | Read-only codebase exploration |
| Plan agent | Task decomposition and planning (no file writes) |
| Verification agent | Mandatory post-task verification contract |
| Review agent | Code review with structured output |

### 5. Skill System

Skills are **prompt-native workflow packages** — not just documentation:

```typescript
// SkillTool invocation (reconstructed)
interface SkillInvocation {
  skillName: string;        // e.g. "fix-tests"
  args: Record<string, string>;
}

// Skills contribute to command system at startup
const allCommands = [
  ...builtinCommands,
  ...pluginCommands,
  ...skillCommands,          // Slash commands from installed skills
  ...bundledSkills,
  ...dynamicSkills,          // Discovered at runtime
].filter(isAvailableInCurrentContext);
```

### 6. Hook System

Hooks intercept tool execution at pre/post points via external scripts:

```bash
# Hook configuration (conceptual, from toolHooks.ts analysis)
# Hooks are registered in session config and called as subprocesses

# Pre-tool hook receives JSON on stdin:
{
  "tool": "Bash",
  "input": { "command": "rm -rf dist/" },
  "session": { "cwd": "/project" }
}

# Hook can respond with:
{
  "decision": "block",         # or "allow", "modify"
  "reason": "Destructive operation requires approval",
  "modifiedInput": null
}
```

### 7. MCP Integration

MCP operates as a **dual integration plane** — tools AND behavior instructions:

```typescript
// MCP contributes both tools and prompt sections
const mcpContext = {
  tools: await mcpClient.listTools(),           // Added to tool registry
  instructions: await mcpClient.getInstructions(), // Injected into system prompt
                                                    // via getMcpInstructionsSection()
};
```

---

## Slash Commands Reference

From `extracted-source/src/commands.ts`:

```
/mcp              Manage MCP server connections
/memory           View/edit persistent memory
/permissions      Show/modify tool permission grants
/hooks            Configure pre/post tool hooks
/plugin           Manage installed plugins
/reload-plugins   Hot-reload plugin definitions
/skills           List available skills
/tasks            View async agent tasks
/plan             Switch to Plan agent mode
/review           Invoke Review agent
/status           Current session state
/model            Switch model
/output-style     Change output verbosity
/agents           List active subagents
/sandbox-toggle   Enable/disable sandbox execution
```

---

## Key Behaviors Documented in the Research

### Tool Grammar Rules (from `getUsingYourToolsSection()`)

```
✅ FileRead   — reading files  (NOT: cat, head, tail, sed)
✅ FileEdit   — editing files  (NOT: sed, awk)
✅ FileWrite  — new files      (NOT: echo > file)
✅ Glob       — finding files  (NOT: find)
✅ Grep       — searching text (NOT: grep via Bash)
✅ Bash       — only when shell semantics are genuinely needed
✅ TodoWrite  — task tracking when available
✅ Parallel   — tool calls with no dependency between them
```

### Engineering Behavior Constraints (from `getSimpleDoingTasksSection()`)

The system prompt explicitly prohibits:
- Adding features the user didn't request
- Over-abstracting or premature generalization
- Unnecessary comments / docstrings / type annotations
- Creating new files when editing existing ones suffices
- Providing time estimates
- Assuming tests passed without running them
- Leaving compatibility shims / dead code after refactors

### Risk Action Rules (from `getActionsSection()`)

Requires confirmation before:
- Destructive or hard-to-reverse operations
- Modifying shared state
- Externally visible actions (git push, deploys, uploads)

---

## Extracting Your Own Source Files

If you want to reproduce the extraction from the npm package:

```bash
# Install the package
npm install @anthropic-ai/claude-code

# The source map is inside the package
ls node_modules/@anthropic-ai/claude-code/
# cli.js   cli.js.map   ...

# Extract sourcesContent from the map (Node.js)
node << 'EOF'
const fs = require('fs');
const map = JSON.parse(fs.readFileSync(
  'node_modules/@anthropic-ai/claude-code/cli.js.map', 'utf8'
));

console.log('Total source files:', map.sources.length);

map.sources.forEach((sourcePath, i) => {
  const content = map.sourcesContent[i];
  if (!content) return;

  const outPath = sourcePath.replace('../', 'extracted-source/');
  fs.mkdirSync(require('path').dirname(outPath), { recursive: true });
  fs.writeFileSync(outPath, content);
});

console.log('Extraction complete');
EOF
```

---

## Reading the Research Report

The PDF is structured as a single long-form research document. Navigation guide:

| Section | What to read it for |
|---|---|
| §1 Research Scope | What was studied, key confirmed facts |
| §2 Source Structure | Why it's an Agent OS, not a CLI wrapper |
| §3 `prompts.ts` | How the main system prompt works |
| §4 Prompt Extraction | Full section-by-section prompt breakdown |
| §5 Agent Prompts | Built-in agent roles and their prompts |
| §6 Agent Dispatch Chain | `AgentTool → runAgent → query` flow |
| §7 Skills/Plugins/Hooks/MCP | The extension ecosystem |
| §8 Permission + Tool Execution | The full execution pipeline |
| §9 Why It's Strong | The architectural moat analysis |
| §10 File Index | Where to look for specific systems |

---

## Common Research Patterns

### "I want to understand how X works"

```bash
# Find the relevant source file
find extracted-source/src -name "*.ts" | xargs grep -l "X" | head -20

# Read the report section
# §3 for prompt assembly
# §6 for agent dispatch
# §7 for skills/hooks/MCP
# §8 for tool execution
```

### "I want to see the actual prompt text"

```bash
# System prompt sections
cat extracted-source/src/constants/prompts.ts

# Agent tool prompt
cat extracted-source/src/tools/AgentTool/prompt.ts

# Skill tool prompt
cat extracted-source/src/tools/SkillTool/prompt.ts
```

### "I want to understand how a specific tool works"

```bash
# All tool implementations
ls extracted-source/src/tools/

# Example: how FileEdit works
cat extracted-source/src/tools/FileEdit.ts

# How tools are executed
cat extracted-source/src/services/tools/toolExecution.ts
cat extracted-source/src/services/tools/toolHooks.ts
```

### "I want to understand the MCP integration"

```bash
cat extracted-source/src/entrypoints/mcp.ts
ls extracted-source/src/services/mcp/

# MCP instructions end up in the system prompt via:
# getMcpInstructionsSection() in prompts.ts
```

---

## Core Thesis (from the Report)

> Claude Code's value is not a single prompt — it is a complete Agent Operating System that unifies prompt assembly, tool execution, permissions, agents, skills, plugins, hooks, MCP, cache economics, and product experience into one coherent runtime.

The architectural moat has six layers:

1. **Prompt assembly with cache economics** — static/dynamic boundary, modular sections
2. **Tool grammar enforcement** — preferred tools for each operation type
3. **Multi-agent specialization** — explore/plan/verify/review agents with scoped tool access
4. **Extension plane** — skills, plugins, hooks, MCP all integrated at runtime
5. **Permission governance** — every tool call is gated and auditable
6. **Behavior constraints** — engineering norms encoded as system prompt policy

---

## Troubleshooting

**PDF won't open / is corrupted**
```bash
# Re-download directly
wget https://github.com/tvytlx/claude-code-deep-dive/raw/main/claude-code-deep-dive-xelatex.pdf
```

**Extracted source files are missing**
```bash
# Check if git-lfs is needed or re-extract from npm
git lfs pull   # if the repo uses LFS for large files
# or re-extract using the Node.js script above
```

**Source file content doesn't match report quotes**
The report was written against a specific version of `@anthropic-ai/claude-code`. The npm package updates frequently — source map content will differ across versions. Pin to the version used in the research if exact reproduction is needed.

**Finding a specific function mentioned in the report**
```bash
# All function names from the report are searchable in extracted source
grep -r "getSessionSpecificGuidanceSection\|SYSTEM_PROMPT_DYNAMIC_BOUNDARY\|runAgent\|toolExecution" \
  extracted-source/src/ --include="*.ts" -l
```
```
