```markdown
---
name: collection-claude-code-source-code
description: A collection and analysis of Claude Code open source artifacts, including decompiled TypeScript source (v2.1.88) and a Python clean-room rewrite reference
triggers:
  - explore claude code source code
  - understand how claude code works internally
  - study claude code architecture
  - analyze claude code tool system
  - learn claude code agent loop
  - implement claude code patterns
  - reference claude code slash commands
  - build something like claude code
---

# Collection: Claude Code Source Code

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

This repository archives and analyzes the internals of Anthropic's Claude Code CLI tool (`@anthropic-ai/claude-code@2.1.88`), providing ~163,318 lines of reconstructed TypeScript source plus a clean-room Python rewrite for study. Use this collection to understand how a production AI coding agent is architected, how its tool system works, and how to replicate its patterns in your own projects.

---

## Repository Structure

```
collection-claude-code-source-code/
├── claude-code-source-code/   # Decompiled TypeScript source (v2.1.88)
│   └── src/
│       ├── main.tsx           # CLI entry + REPL bootstrap (~4,683 lines)
│       ├── query.ts           # Core agent loop (~785KB)
│       ├── QueryEngine.ts     # SDK/Headless query lifecycle
│       ├── Tool.ts            # Tool interface + buildTool factory
│       ├── commands.ts        # Slash command definitions (~25K lines)
│       ├── tools/             # 40+ tool implementations
│       ├── components/        # React/Ink terminal UI
│       ├── services/          # Business logic layer
│       ├── coordinator/       # Multi-agent coordination
│       ├── memdir/            # Long-term memory management
│       └── plugins/           # Plugin system
├── claw-code/                 # Clean-room Python rewrite (66 files)
└── docs/                      # Analysis docs (English + Chinese)
    ├── en/
    └── zh/
```

---

## Installation / Setup

```bash
# Clone the collection
git clone https://github.com/chauncygu/collection-claude-code-source-code.git
cd collection-claude-code-source-code

# Install the actual Claude Code CLI (official)
npm install -g @anthropic-ai/claude-code

# Or explore the TypeScript source directly
cd claude-code-source-code
npm install
```

Set your Anthropic API key before using Claude Code:

```bash
export ANTHROPIC_API_KEY=your_key_here
```

---

## Core Architecture Patterns

### Agent Loop (query.ts)

The central loop drives all agent behavior. Key stages:

```typescript
// Simplified representation of the main agent loop pattern from query.ts
async function* query(
  input: UserInput,
  context: AgentContext
): AsyncGenerator<SDKMessage> {
  // 1. Assemble system prompt
  const systemPromptParts = await fetchSystemPromptParts(context);

  // 2. Stream Claude API response
  const stream = client.messages.stream({
    model: context.model,
    system: systemPromptParts.join('\n'),
    messages: context.history,
    tools: context.tools,
  });

  // 3. Execute tools in parallel as they arrive
  const executor = new StreamingToolExecutor();
  for await (const event of stream) {
    if (event.type === 'tool_use') {
      yield* executor.run(event, context);
    } else {
      yield event;
    }
  }

  // 4. Auto-compact context if token limit approaches
  await autoCompact(context);
}
```

### Tool Interface (Tool.ts)

Every tool follows this contract:

```typescript
interface Tool<TInput, TOutput> {
  name: string;
  description: string;
  inputSchema: ZodSchema<TInput>;
  call(input: TInput, context: ToolContext): Promise<TOutput>;
  isReadOnly?: boolean;
  requiresPermission?: boolean;
}

// Build a tool using the factory
const MyFileTool = buildTool({
  name: 'read_file',
  description: 'Read a file from disk',
  inputSchema: z.object({
    path: z.string().describe('Absolute path to file'),
  }),
  async call({ path }, context) {
    const content = await fs.readFile(path, 'utf-8');
    return { content, lines: content.split('\n').length };
  },
});
```

### Implementing a Custom Tool (TypeScript)

```typescript
import { z } from 'zod';
import { buildTool } from './src/Tool';

export const WebhookTool = buildTool({
  name: 'send_webhook',
  description: 'Send a POST request to a webhook URL with a JSON payload',
  inputSchema: z.object({
    url: z.string().url(),
    payload: z.record(z.unknown()),
  }),
  isReadOnly: false,
  requiresPermission: true,
  async call({ url, payload }, _context) {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });
    return {
      status: res.status,
      ok: res.ok,
      body: await res.text(),
    };
  },
});
```

---

## Key Tool Categories

| Category | Tools |
|---|---|
| File Operations | `FileReadTool`, `FileEditTool`, `FileWriteTool` |
| Code Search | `GlobTool`, `GrepTool` |
| System Execution | `BashTool` |
| Web Access | `WebFetchTool`, `WebSearchTool` |
| Task Management | `TaskCreateTool`, `TaskUpdateTool`, `TaskGetTool`, `TaskListTool` |
| Sub-agents | `AgentTool` |
| Code Environments | `NotebookEditTool`, `REPLTool`, `LSPTool` |
| Git Workflow | `EnterWorktreeTool`, `ExitWorktreeTool` |
| Memory & Planning | `TodoWriteTool`, `EnterPlanModeTool`, `ExitPlanModeTool` |
| Automation | `ScheduleCronTool`, `RemoteTriggerTool`, `SleepTool` |
| MCP Integration | `MCPTool` |

---

## Permission System

Three enforcement modes control tool access:

```typescript
type PermissionMode = 'default' | 'bypass' | 'strict';

// default  → ask the user before executing sensitive tools
// bypass   → auto-allow all tools (headless/CI use)
// strict   → auto-deny all tools requiring permissions

// Example: configuring permissions in headless mode
const context: AgentContext = {
  permissionMode: 'bypass',  // useful in CI pipelines
  toolPermissions: {
    BashTool: 'allow',
    FileWriteTool: 'ask',
    WebFetchTool: 'deny',
  },
};
```

---

## Context Compression Strategies

Claude Code uses three auto-compaction strategies when context grows large:

```typescript
// From autoCompact() in query.ts — simplified
async function autoCompact(context: AgentContext) {
  const tokenCount = estimateTokens(context.history);

  if (tokenCount > REACTIVE_THRESHOLD) {
    // Strategy 1: Reactive — summarize old turns
    context.history = await summarizeOldTurns(context.history);
  }

  if (tokenCount > MICRO_THRESHOLD) {
    // Strategy 2: Micro — strip whitespace + comments
    context.history = microCompress(context.history);
  }

  if (tokenCount > COLLAPSE_THRESHOLD) {
    // Strategy 3: Collapse — keep only key tool results
    context.history = collapseContext(context.history);
  }
}
```

---

## Slash Commands Reference

Claude Code exposes ~87 slash commands. Key ones:

| Command | Purpose |
|---|---|
| `/commit` | Stage and commit changes |
| `/commit-push-pr` | Commit, push, and open a PR |
| `/review` | AI code review of current diff |
| `/resume` | Resume a previous session |
| `/session` | Manage session state |
| `/memory` | View/edit long-term memory |
| `/config` | Adjust configuration |
| `/skills` | Manage installed skills |
| `/permissions` | Manage tool permissions |
| `/mcp` | Model Context Protocol tools |
| `/vim` | Toggle Vim keybindings |
| `/voice` | Toggle voice mode |

### Implementing a Custom Slash Command

```typescript
// Pattern from src/commands/ directory
export const myCommand = {
  name: 'deploy',
  description: 'Deploy the current project to production',
  aliases: ['/deploy'],
  async execute(args: string[], context: CommandContext) {
    const env = args[0] ?? 'staging';
    await context.runTool('BashTool', {
      command: `./scripts/deploy.sh ${env}`,
    });
    return `Deployed to ${env}`;
  },
};
```

---

## Memory System (memdir/)

Claude Code implements 7 layers of memory:

```typescript
// Long-term memory stored in ~/.claude/memory/
// Key patterns from memdir/ module:

// Write to long-term memory
await memoryStore.set('project_context', {
  language: 'TypeScript',
  framework: 'Next.js',
  conventions: ['use named exports', 'prefer async/await'],
});

// Read from memory in system prompt assembly
const memory = await memoryStore.getAll();
const memoryBlock = formatMemoryForPrompt(memory);
```

---

## Multi-Agent Coordination (coordinator/)

```typescript
// Pattern for spawning sub-agents (AgentTool)
const subAgentResult = await agentTool.call({
  task: 'Write unit tests for src/utils/parser.ts',
  tools: ['FileReadTool', 'FileWriteTool', 'BashTool'],
  maxTurns: 10,
});

// The coordinator manages:
// - Task queuing and scheduling
// - Result aggregation
// - Shared context between agents
// - Token budget allocation per sub-agent
```

---

## Python Rewrite (claw-code) Reference

The `claw-code` subproject provides a minimal Python reimplementation. Key classes:

```python
# claw-code architecture overview
from claw_code import ClawAgent, Tool, PermissionMode

agent = ClawAgent(
    api_key=os.environ["ANTHROPIC_API_KEY"],
    model="claude-opus-4-5",
    permission_mode=PermissionMode.DEFAULT,
)

# Register a tool
@agent.tool(requires_permission=True)
async def run_bash(command: str) -> str:
    """Execute a shell command."""
    result = await asyncio.create_subprocess_shell(command)
    stdout, _ = await result.communicate()
    return stdout.decode()

# Run the agent loop
async def main():
    async for message in agent.query("Fix the failing tests in ./tests/"):
        print(message)
```

---

## Tech Stack Summary

| Component | Technology |
|---|---|
| Language | TypeScript 6.0+ |
| Runtime | Bun (compiled to Node.js ≥ 18 bundle) |
| LLM API | Anthropic SDK |
| Terminal UI | React + Ink |
| Bundler | esbuild |
| Validation | Zod |
| Tool Protocol | MCP (Model Context Protocol) |
| Python rewrite | Python 3.11+, asyncio |

---

## Common Patterns for Agent Builders

### Streaming Tool Results

```typescript
// Pattern: yield partial results as tools execute
async function* runTools(
  toolCalls: ToolUseBlock[],
  context: AgentContext
): AsyncGenerator<ToolResult> {
  const results = await Promise.allSettled(
    toolCalls.map(tc => context.tools[tc.name].call(tc.input, context))
  );

  for (const [i, result] of results.entries()) {
    yield {
      tool_use_id: toolCalls[i].id,
      type: 'tool_result',
      content: result.status === 'fulfilled'
        ? JSON.stringify(result.value)
        : `Error: ${result.reason}`,
      is_error: result.status === 'rejected',
    };
  }
}
```

### System Prompt Assembly

```typescript
// Pattern from fetchSystemPromptParts()
async function fetchSystemPromptParts(ctx: AgentContext): Promise<string[]> {
  return [
    CORE_IDENTITY_PROMPT,
    await getProjectContext(ctx.cwd),       // CLAUDE.md / project instructions
    await getMemoryBlock(ctx.userId),        // Long-term memory
    await getInstalledSkills(ctx.skillsDir), // Installed SKILL.md files
    getToolDescriptions(ctx.tools),          // Available tools
    getPermissionInstructions(ctx.mode),     // Permission rules
  ];
}
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `ANTHROPIC_API_KEY` not found | Set `export ANTHROPIC_API_KEY=sk-ant-...` in your shell |
| TypeScript source won't compile | Requires TypeScript 6.0+ and Bun runtime |
| Context window exceeded | Implement `autoCompact()` pattern; monitor token counts |
| Tool permission denied in CI | Set `permissionMode: 'bypass'` for headless environments |
| Sub-agent loop hangs | Set `maxTurns` limit on `AgentTool` invocations |
| Memory not persisting | Check write permissions on `~/.claude/memory/` directory |

---

## Further Reading

- `docs/en/` — English architecture analysis documents
- `docs/zh/` — Chinese deep-dive analyses
- `docs/claude-code-deep-dive-xelatex.pdf` — Full research report
- [nano-claude-code](https://github.com/SafeRL-Lab/nano-claude-code) — ~3,400 line minimal Python reimplementation
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47586778) — Community analysis
```
