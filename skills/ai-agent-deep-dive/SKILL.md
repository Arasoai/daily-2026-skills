```markdown
---
name: ai-agent-deep-dive
description: Research notes and analysis on modern Coding Agent architecture — covers prompt systems, agent orchestration, skills, plugins, hooks, MCP, and tool execution pipelines as seen in Claude Code.
triggers:
  - help me understand how Claude Code works internally
  - explain the architecture of a coding agent
  - how does agent orchestration work in Claude Code
  - what is the MCP integration pattern for agents
  - how do skills and plugins work in coding agents
  - explain the system prompt assembly for AI agents
  - how does tool permission and hook execution work
  - I want to build a coding agent like Claude Code
---

# AI Agent Deep Dive

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A structured research report and analysis of modern Coding Agent architecture, using Claude Code as the primary reference. This repository contains a PDF report and annotated notes covering prompt engineering, agent orchestration, tool execution pipelines, permission models, and extensibility systems (Skills, Plugins, Hooks, MCP).

---

## What This Project Is

This is **not a runnable library** — it is a **deep-dive research document** (`ai-agent-deep-dive-report.pdf`) analyzing how a mature Coding Agent (Claude Code) works from an architectural perspective. It is useful for:

- Developers building their own coding agents
- Teams designing agent orchestration systems
- Engineers integrating MCP, hooks, or tool pipelines
- Researchers studying production-grade LLM agent systems

The core thesis: *Claude Code's strength is not a clever system prompt — it is a complete Agent Operating System.*

---

## How to Access the Report

```bash
# Clone the repository
git clone https://github.com/tvytlx/ai-agent-deep-dive.git
cd ai-agent-deep-dive

# Open the PDF report (primary artifact)
open ai-agent-deep-dive-report.pdf
# or
xdg-open ai-agent-deep-dive-report.pdf  # Linux
```

The README itself also contains the full annotated notes inline — no build step needed.

---

## Core Architecture Concepts Covered

### 1. Agent Operating System Mental Model

A mature coding agent is structured as a platform, not a script:

```
src/
├── entrypoints/        # cli.tsx, init.ts, mcp.ts, sdk/
├── constants/          # prompts.ts — system prompt assembly
├── tools/              # FileRead, FileEdit, Bash, Agent, Skill, MCP...
├── services/           # tools, mcp, analytics runtime services
├── commands/           # slash commands (/mcp, /hooks, /skills, /plan...)
├── coordinator/        # agent coordination layer
├── plugins/            # plugin ecosystem
├── hooks/              # hook system
├── tasks/              # local, remote, async agent tasks
├── memdir/             # memory/prompt injection
└── bootstrap/          # state initialization
```

Key insight: the same agent runtime serves CLI, MCP mode, and SDK consumers simultaneously.

---

### 2. System Prompt Assembly Pattern

The system prompt is **not a static string** — it is a runtime-assembled module chain:

```typescript
// Conceptual reconstruction of getSystemPrompt() architecture
function getSystemPrompt(session: SessionContext): string {
  // --- STATIC PREFIX (cache-friendly) ---
  const staticSections = [
    getSimpleIntroSection(),        // identity + role
    getSimpleSystemSection(),       // base rules
    getSimpleDoingTasksSection(),   // task philosophy
    getActionsSection(),            // allowed actions
    getUsingYourToolsSection(),     // tool usage norms
    getSimpleToneAndStyleSection(), // communication style
    getOutputEfficiencySection(),   // token hygiene
  ].join("\n\n");

  // --- DYNAMIC SUFFIX (session-specific) ---
  const dynamicSections = [
    session.guidance     ? getSessionGuidance(session)     : "",
    session.memory       ? getMemoryPrompt(session)        : "",
    getEnvInfoSection(session.env),
    session.language     ? getLanguageSection(session)     : "",
    session.outputStyle  ? getOutputStyleSection(session)  : "",
    session.mcpServers   ? getMCPInstructions(session)     : "",
    getScratchpadSection(),
    getFunctionResultClearingPrompt(),
    session.tokenBudget  ? getTokenBudgetSection(session)  : "",
    session.brief        ? getBriefModeSection()           : "",
  ].filter(Boolean).join("\n\n");

  return [staticSections, dynamicSections].join("\n\n");
}
```

**Why this matters:** Static sections are cache-stable (cheaper), dynamic sections adapt per session. This is prompt architecture, not prompt writing.

---

### 3. Tool Execution Pipeline

Tools are never called directly — every invocation goes through a governance pipeline:

```typescript
// Conceptual tool execution pipeline
async function executeTool(toolCall: ToolCall, context: AgentContext) {
  // 1. Schema validation
  const parsed = toolSchema.parse(toolCall.input);

  // 2. Input validation (tool-specific)
  const validationResult = await tool.validateInput(parsed, context);
  if (!validationResult.ok) throw new ValidationError(validationResult.error);

  // 3. Pre-tool hooks (can modify input, inject context, or BLOCK)
  const hookDecision = await runPreToolHooks(toolCall, context);
  if (hookDecision.action === "block") {
    return { blocked: true, reason: hookDecision.reason };
  }

  // 4. Permission check
  const permission = await checkPermission(toolCall, context);
  if (!permission.granted) {
    return await requestUserPermission(toolCall, context);
  }

  // 5. Actual tool execution
  const result = await tool.execute(parsed, context);

  // 6. Telemetry / analytics
  await recordToolUsage(toolCall, result, context);

  // 7. Post-tool hooks
  const finalResult = await runPostToolHooks(toolCall, result, context);

  return finalResult;
}
```

**Available built-in tools:**

| Tool | Purpose |
|---|---|
| `FileRead` | Read file contents |
| `FileEdit` | Patch/edit existing files |
| `FileWrite` | Create or overwrite files |
| `Bash` | Execute shell commands |
| `Glob` | File pattern matching |
| `Grep` | Content search |
| `TodoWrite` | Task tracking |
| `TaskCreate` | Async agent task creation |
| `AskUserQuestion` | Clarify ambiguity |
| `Skill` | Invoke a packaged workflow |
| `Agent` | Spawn a subagent |
| `MCPTool` | Call an MCP-registered tool |
| `Sleep` | Delay execution |

---

### 4. Agent Orchestration and Subagents

The `AgentTool` is how the main agent spawns specialized subagents:

```typescript
// Conceptual AgentTool dispatch logic
async function agentToolHandler(input: AgentToolInput, ctx: AgentContext) {
  const mode = resolveAgentMode(input, ctx);
  // mode: "fork" | "normal" | "background" | "remote" | "worktree"

  const subagentPrompt = buildSubagentPrompt(input.task, mode);
  const tools = selectToolsForMode(mode, ctx);
  const systemPrompt = getSystemPrompt(buildSubagentSession(ctx, mode));

  return await runAgent({
    messages: subagentPrompt,
    systemPrompt,
    tools,
    context: ctx,
  });
}
```

**Built-in specialized agents:**

```
General Purpose Agent  → default task execution
Explore Agent          → codebase discovery, read-only recon
Plan Agent             → structured planning before execution  
Verification Agent     → post-implementation validation
```

**Verification Agent pattern** (especially valuable):

```typescript
// What Verification Agent checks:
const verificationChecklist = [
  "npm run build",          // does it compile?
  "npm test",               // do tests pass?
  "npx tsc --noEmit",       // type errors?
  // ... real command outputs, not assumptions
  "final verdict: PASS | FAIL with specific reasons"
];
```

---

### 5. Hook System

Hooks provide runtime governance — they can observe, modify, or block agent behavior:

```typescript
// Hook interface pattern
interface AgentHook {
  name: string;
  event: "pre-tool" | "post-tool" | "on-failure" | "on-permission-request";
  handler: (context: HookContext) => Promise<HookDecision>;
}

interface HookDecision {
  action: "allow" | "block" | "modify" | "inject-context";
  modifiedInput?: unknown;
  injectedContext?: string;
  reason?: string;
}

// Example: a hook that blocks file writes outside the project root
const safeWriteHook: AgentHook = {
  name: "safe-write-guard",
  event: "pre-tool",
  handler: async (ctx) => {
    if (ctx.toolName !== "FileWrite") return { action: "allow" };
    const targetPath = ctx.toolInput.path as string;
    if (!targetPath.startsWith(ctx.projectRoot)) {
      return {
        action: "block",
        reason: `Write outside project root blocked: ${targetPath}`,
      };
    }
    return { action: "allow" };
  },
};
```

---

### 6. Skill System

Skills are **reusable prompt-native workflow packages**, not documentation:

```typescript
// Skill definition pattern
interface Skill {
  name: string;
  description: string;
  triggerPhrases: string[];
  workflowPrompt: string;       // injected into context when invoked
  requiredTools: string[];
  metadata: {
    author: string;
    version: string;
    tags: string[];
  };
}

// Example skill: "write-tests"
const writeTestsSkill: Skill = {
  name: "write-tests",
  description: "Write comprehensive tests for a given module",
  triggerPhrases: ["write tests for", "add test coverage to"],
  workflowPrompt: `
    When writing tests:
    1. First read the target file completely
    2. Identify all exported functions and their edge cases
    3. Check existing test patterns in __tests__/ or *.test.ts files
    4. Write tests that cover: happy path, error cases, edge cases
    5. Run the tests and fix any failures before reporting done
  `,
  requiredTools: ["FileRead", "FileWrite", "Bash", "Glob"],
  metadata: { author: "team", version: "1.0.0", tags: ["testing"] },
};
```

---

### 7. MCP Integration Pattern

MCP (Model Context Protocol) is not just a tool bridge — it also injects behavioral instructions:

```typescript
// MCP server registration and prompt injection
interface MCPServer {
  name: string;
  transport: "stdio" | "sse" | "http";
  command?: string;   // for stdio
  url?: string;       // for sse/http
  instructions?: string;  // injected into system prompt!
  tools: MCPToolDefinition[];
}

// claude_desktop_config.json / .mcp.json pattern
const mcpConfig = {
  mcpServers: {
    "filesystem": {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/project"],
      instructions: "Use this for all file operations on /project"
    },
    "github": {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: {
        GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"  // env var ref
      }
    }
  }
};
```

---

### 8. Permission Model

Permissions follow a layered decision model:

```typescript
type PermissionLevel =
  | "always-allow"      // pre-approved, no prompt
  | "ask-once"          // ask user, remember for session
  | "ask-every-time"    // always prompt
  | "always-deny";      // blocked by policy

interface PermissionPolicy {
  tool: string;
  pattern?: string;     // e.g., path glob for FileWrite
  level: PermissionLevel;
  scope: "session" | "project" | "global";
}

// Example policy config (CLAUDE.md or settings.json)
const permissionPolicies: PermissionPolicy[] = [
  { tool: "FileRead",  level: "always-allow", scope: "session" },
  { tool: "Bash",      pattern: "npm test*",  level: "always-allow", scope: "project" },
  { tool: "Bash",      pattern: "rm -rf*",    level: "always-deny",  scope: "global" },
  { tool: "FileWrite", pattern: "src/**",     level: "ask-once",     scope: "session" },
];
```

---

## Key Design Principles (From the Research)

### Task Philosophy Encoded in Prompt

```
✅ Read code before changing it
✅ Report results honestly
✅ Do exactly what was asked

❌ Add unrequested features
❌ Over-abstract prematurely  
❌ Refactor without being asked
❌ Create new files unnecessarily
❌ Retry after user rejection
```

### Context Hygiene

```typescript
// Function result clearing pattern — prevents context bloat
const FUNCTION_RESULT_CLEARING_PROMPT = `
  When tool results are no longer needed for the current task,
  they can be summarized to reduce context window pressure.
  Prioritize keeping: recent results, error messages, file contents
  currently being edited.
`;
```

---

## Building Your Own Coding Agent: Checklist

Based on this research, a production-grade coding agent needs:

```
Architecture
├── [ ] Modular system prompt assembly (static + dynamic sections)
├── [ ] Multiple entry points (CLI, SDK, MCP server mode)
└── [ ] Platform-level command system

Tool Runtime
├── [ ] Schema validation before execution
├── [ ] Pre/post tool hooks
├── [ ] Permission model with scoping
└── [ ] Telemetry per tool call

Agent Orchestration  
├── [ ] Specialized built-in agents (explore, plan, verify)
├── [ ] Fork/subagent dispatch logic
└── [ ] Background/async task lifecycle

Extensibility
├── [ ] Skill system (workflow packages)
├── [ ] Plugin system (prompt + metadata + constraints)
├── [ ] Hook system (governance layer)
└── [ ] MCP integration (tools + behavioral instructions)

Context Management
├── [ ] Memory/memdir injection
├── [ ] Token budget awareness
├── [ ] Function result summarization
└── [ ] Prompt injection defense
```

---

## Slash Commands Reference

The command system is the user's control surface for the agent runtime:

| Command | Purpose |
|---|---|
| `/mcp` | Manage MCP server connections |
| `/memory` | View/edit agent memory |
| `/permissions` | Inspect/modify permission policies |
| `/hooks` | View/configure runtime hooks |
| `/plugin` | Manage plugins |
| `/reload-plugins` | Hot-reload plugin definitions |
| `/skills` | List and invoke available skills |
| `/tasks` | View async/background task status |
| `/plan` | Enter planning mode before execution |
| `/review` | Trigger verification agent |
| `/status` | Current agent state |
| `/model` | Switch underlying model |
| `/output-style` | Adjust verbosity/format |
| `/agents` | View active subagent tree |
| `/sandbox-toggle` | Enable/disable sandboxed execution |

---

## Troubleshooting Common Agent Design Problems

**Problem: Agent keeps retrying after user says no**
→ Encode in system prompt: "If user declines, do not retry with same approach"
→ Add a pre-tool hook that checks for recent user rejections

**Problem: Agent over-engineers solutions**
→ Add explicit prompt section: no unrequested abstractions, no premature refactoring

**Problem: Context window fills up with tool results**
→ Implement function result clearing / summarization
→ Use token budget section in system prompt

**Problem: Subagents don't know what tools they have**
→ Pass tool list explicitly when constructing subagent system prompt
→ Include `getUsingYourToolsSection()` equivalent in subagent prompt

**Problem: MCP tools used incorrectly**
→ Use MCP `instructions` field to inject behavioral guidance alongside tool definitions

**Problem: Unsafe bash commands executed**
→ Add pre-tool hooks for Bash with pattern matching
→ Set `always-deny` permission policies for destructive patterns

---

## Resources

- **Primary artifact:** `ai-agent-deep-dive-report.pdf` in the repository root
- **Repository:** https://github.com/tvytlx/ai-agent-deep-dive
- **MCP Specification:** https://modelcontextprotocol.io
- **Claude Code docs:** https://docs.anthropic.com/claude-code
```
