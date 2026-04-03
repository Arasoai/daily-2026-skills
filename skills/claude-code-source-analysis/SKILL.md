```markdown
---
name: claude-code-source-analysis
description: Expertise in exploring, understanding, and extending the Claude Code decompiled source archive and its Python reimplementation (claw-code)
triggers:
  - explore claude code source code
  - understand claude code architecture
  - analyze claude code internals
  - work with claude code tools system
  - study claude code agent loop
  - implement claude code patterns
  - extend claude code slash commands
  - understand claude code memory system
---

# Claude Code Source Code Collection

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection

A research repository containing the decompiled TypeScript source of Claude Code v2.1.88 (~163,318 lines across 1,884 files) and a clean-room Python reimplementation (`claw-code`). Use this repository to study, extend, or reimplement Anthropic's CLI coding agent.

---

## Repository Structure

```
collection-claude-code-source-code/
├── claude-code-source-code/   # Decompiled TypeScript (v2.1.88)
│   └── src/
│       ├── main.tsx           # CLI entry + REPL bootstrap
│       ├── query.ts           # Core agent loop (785KB)
│       ├── QueryEngine.ts     # SDK/Headless lifecycle engine
│       ├── Tool.ts            # Tool interface + buildTool factory
│       ├── commands.ts        # Slash command definitions
│       ├── tools/             # 40+ tool implementations
│       ├── commands/          # ~87 slash command handlers
│       ├── components/        # React/Ink terminal UI
│       ├── services/          # Business logic layer
│       ├── coordinator/       # Multi-agent coordination
│       ├── memdir/            # Long-term memory management
│       └── plugins/           # Plugin system
├── claw-code/                 # Python clean-room rewrite (66 files)
└── docs/                      # Bilingual analysis (en/ + zh/)
```

---

## Installation & Setup

```bash
# Clone the repository
git clone https://github.com/chauncygu/collection-claude-code-source-code.git
cd collection-claude-code-source-code

# Explore the TypeScript source
cd claude-code-source-code
npm install   # if package.json dependencies are needed for tooling

# Work with the Python rewrite
cd ../claw-code
pip install -r requirements.txt
```

---

## Core Architecture: The Agent Loop

The central execution model lives in `src/query.ts`. Understanding it is key to understanding the whole system.

```typescript
// Simplified agent loop pattern from query.ts
async function* query(
  userMessage: string,
  context: ConversationContext,
  tools: Tool[],
): AsyncGenerator<SDKMessage> {
  // 1. Assemble system prompt from parts
  const systemPromptParts = await fetchSystemPromptParts(context);

  // 2. Run streaming tool executor with auto-compaction
  const executor = new StreamingToolExecutor(tools);

  while (true) {
    const response = await callClaude({ systemPromptParts, userMessage, tools });

    // 3. Yield streamed messages back to consumer
    for await (const chunk of response) {
      yield chunk;
    }

    // 4. Execute tool calls in parallel
    const toolResults = await executor.runTools(response.toolCalls);

    // 5. Auto-compact context if approaching token limit
    if (shouldCompact(context)) {
      await autoCompact(context);
    }

    if (!hasMoreToolCalls(response)) break;
  }
}
```

### Entry Point Pattern (`main.tsx`)

```typescript
// CLI bootstrap pattern
import { render } from 'ink';
import { App } from './components/App';

async function main() {
  const args = parseArgs(process.argv.slice(2));

  if (args.headless) {
    // SDK/headless mode via QueryEngine
    const engine = new QueryEngine(args);
    await engine.run();
  } else {
    // Interactive REPL mode via React/Ink
    render(<App initialArgs={args} />);
  }
}

main().catch(console.error);
```

---

## Tool System

### Tool Interface (`src/Tool.ts`)

```typescript
interface Tool {
  name: string;
  description: string;
  inputSchema: ZodSchema;
  execute(input: unknown, context: ToolContext): Promise<ToolResult>;
}

// buildTool factory pattern
const MyTool = buildTool({
  name: 'my_tool',
  description: 'Does something useful',
  inputSchema: z.object({
    path: z.string().describe('File path to operate on'),
    content: z.string().optional(),
  }),
  async execute({ path, content }, ctx) {
    // tool implementation
    return { type: 'text', text: `Processed ${path}` };
  },
});
```

### Key Built-in Tools

```typescript
// File operations
import { FileReadTool }  from './tools/FileReadTool';
import { FileEditTool }  from './tools/FileEditTool';
import { FileWriteTool } from './tools/FileWriteTool';

// Code search
import { GlobTool } from './tools/GlobTool';
import { GrepTool } from './tools/GrepTool';

// Execution
import { BashTool } from './tools/BashTool';

// Web
import { WebFetchTool }  from './tools/WebFetchTool';
import { WebSearchTool } from './tools/WebSearchTool';

// Sub-agents
import { AgentTool } from './tools/AgentTool';

// Memory
import { TodoWriteTool } from './tools/TodoWriteTool';
```

### Registering Tools (`src/tools.ts`)

```typescript
// Tool registration pattern
export function getTools(config: Config): Tool[] {
  const baseTools = [
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    BashTool,
  ];

  if (config.enableWebTools) {
    baseTools.push(WebFetchTool, WebSearchTool);
  }

  if (config.enableAgentTools) {
    baseTools.push(AgentTool);
  }

  return baseTools;
}
```

---

## Slash Commands

### Command Definition Pattern (`src/commands/`)

```typescript
// Pattern for defining a slash command
export const reviewCommand = {
  name: 'review',
  description: 'Review code changes',
  aliases: ['/review'],
  async execute(args: string[], context: CommandContext) {
    const diff = await context.tools.bash.execute('git diff HEAD');
    return context.query(`Please review these changes:\n${diff}`);
  },
};

// Registration in commands.ts
export const SLASH_COMMANDS = [
  reviewCommand,
  commitCommand,
  sessionCommand,
  memoryCommand,
  configCommand,
  // ... ~87 total
];
```

### Common Slash Commands Reference

| Command | Purpose |
|---|---|
| `/commit` | Stage and commit changes |
| `/commit-push-pr` | Commit, push, and open PR |
| `/review` | Review current diff |
| `/resume` | Resume last session |
| `/memory` | Manage long-term memory |
| `/config` | Edit configuration |
| `/skills` | List available skills |
| `/permissions` | Manage tool permissions |
| `/mcp` | MCP server management |
| `/vim` | Toggle vim keybindings |

---

## Permission System

```typescript
// Three permission modes
type PermissionMode = 'default' | 'bypass' | 'strict';

// default  → ask user before executing sensitive tools
// bypass   → auto-allow all tools (headless/CI use)
// strict   → auto-deny all unwhitelisted tools

// Permission rule structure
interface PermissionRule {
  tool: string;           // e.g. 'bash', 'file_write'
  pattern?: string;       // glob pattern for path-based rules
  mode: PermissionMode;
}

// Checking permissions at runtime
async function checkPermission(
  tool: Tool,
  input: unknown,
  rules: PermissionRule[],
): Promise<'allow' | 'deny' | 'ask'> {
  const matchingRule = rules.find(r => matchesRule(r, tool, input));
  if (matchingRule) return matchingRule.mode === 'bypass' ? 'allow' : 'deny';
  return 'ask'; // default: prompt user
}
```

---

## Context Management & Auto-Compaction

```typescript
// Auto-compact strategies from query.ts
type CompactionStrategy =
  | 'reactive'   // compress when near token limit
  | 'micro'      // compress small incremental chunks
  | 'trimmed';   // trim oldest turns first

async function autoCompact(
  context: ConversationContext,
  strategy: CompactionStrategy = 'reactive',
): Promise<void> {
  const tokenCount = await estimateTokens(context.messages);

  if (tokenCount > CONTEXT_COLLAPSE_THRESHOLD) {
    const summary = await summarizeHistory(context.messages);
    context.messages = [
      { role: 'system', content: summary },
      ...context.messages.slice(-KEEP_LAST_N_MESSAGES),
    ];
  }
}

// Token estimation utility
function estimateTokens(messages: Message[]): number {
  return messages.reduce((sum, m) => sum + Math.ceil(m.content.length / 4), 0);
}
```

---

## Memory System (`src/memdir/`)

Claude Code implements a 7-layer memory architecture:

```typescript
// Memory layer types
type MemoryLayer =
  | 'working'       // current session context window
  | 'episodic'      // compressed past session summaries
  | 'semantic'      // extracted facts and preferences
  | 'procedural'    // learned workflows and patterns
  | 'external'      // files, docs, codebases
  | 'meta'          // memory about memory (indexing)
  | 'dream';        // background consolidation process

// Writing to long-term memory
import { TodoWriteTool } from './tools/TodoWriteTool';

// Memory entries are stored as structured markdown in ~/.claude/memory/
interface MemoryEntry {
  id: string;
  layer: MemoryLayer;
  content: string;
  embedding?: number[];  // for semantic search
  createdAt: Date;
  tags: string[];
}
```

---

## Multi-Agent Coordination (`src/coordinator/`)

```typescript
// Sub-agent spawning via AgentTool
const result = await AgentTool.execute({
  task: 'Analyze the test failures in src/utils/',
  tools: ['file_read', 'grep', 'bash'],
  context: {
    maxTokens: 8192,
    systemPrompt: 'You are a debugging specialist.',
  },
});

// Coordinator pattern for parallel agents
class AgentCoordinator {
  async runParallel(tasks: AgentTask[]): Promise<AgentResult[]> {
    return Promise.all(tasks.map(task => this.spawnAgent(task)));
  }

  async runSequential(tasks: AgentTask[]): Promise<AgentResult[]> {
    const results = [];
    for (const task of tasks) {
      results.push(await this.spawnAgent(task));
    }
    return results;
  }
}
```

---

## QueryEngine (Headless/SDK Mode)

```typescript
// Using QueryEngine for programmatic access
import { QueryEngine } from './QueryEngine';

const engine = new QueryEngine({
  apiKey: process.env.ANTHROPIC_API_KEY,
  model: 'claude-opus-4-5',
  permissionMode: 'bypass',  // for CI environments
  tools: getTools({ enableWebTools: true }),
});

// Single query
const result = await engine.query('Refactor the auth module to use JWT');

// Streaming
for await (const chunk of engine.queryStream('Write tests for UserService')) {
  process.stdout.write(chunk.text ?? '');
}
```

---

## Python Rewrite: claw-code

```python
# claw-code mirrors the TypeScript architecture in Python
# Core agent loop
from claw_code.agent import ClawAgent
from claw_code.tools import FileReadTool, BashTool, GrepTool

agent = ClawAgent(
    api_key=os.environ["ANTHROPIC_API_KEY"],
    model="claude-opus-4-5",
    tools=[FileReadTool(), BashTool(), GrepTool()],
)

# Run a task
result = agent.run("Find all TODO comments and create a summary report")
print(result)

# Async streaming
async for chunk in agent.stream("Refactor the database layer"):
    print(chunk, end="", flush=True)
```

```python
# Implementing a custom tool in claw-code
from claw_code.tools.base import BaseTool
from pydantic import BaseModel

class MyToolInput(BaseModel):
    path: str
    pattern: str

class MyCustomTool(BaseTool):
    name = "my_custom_tool"
    description = "Search files matching a pattern"
    input_schema = MyToolInput

    def execute(self, input: MyToolInput, context) -> str:
        import subprocess
        result = subprocess.run(
            ["grep", "-r", input.pattern, input.path],
            capture_output=True, text=True
        )
        return result.stdout
```

---

## MCP Integration

```typescript
// Model Context Protocol tool registration
import { MCPTool } from './tools/MCPTool';

// Connect to an MCP server
const mcpTool = new MCPTool({
  serverUrl: process.env.MCP_SERVER_URL,
  capabilities: ['resources', 'tools', 'prompts'],
});

// Tools exposed by MCP servers are auto-discovered
const mcpTools = await mcpTool.discoverTools();
// Returns standard Tool[] that integrate seamlessly
```

---

## React/Ink Terminal UI Patterns

```typescript
// Component pattern from src/components/
import React, { useState } from 'react';
import { Box, Text, useInput } from 'ink';

const AgentOutput: React.FC<{ messages: Message[] }> = ({ messages }) => {
  return (
    <Box flexDirection="column">
      {messages.map((msg, i) => (
        <Box key={i} marginBottom={1}>
          <Text color={msg.role === 'assistant' ? 'cyan' : 'white'}>
            {msg.role === 'assistant' ? '🤖 ' : '👤 '}
            {msg.content}
          </Text>
        </Box>
      ))}
    </Box>
  );
};

// Custom hook for agent state
function useAgentState() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isThinking, setIsThinking] = useState(false);

  async function sendMessage(text: string) {
    setIsThinking(true);
    for await (const chunk of queryStream(text)) {
      setMessages(prev => appendChunk(prev, chunk));
    }
    setIsThinking(false);
  }

  return { messages, isThinking, sendMessage };
}
```

---

## Configuration

```typescript
// Config structure (src/setup.ts)
interface ClaudeCodeConfig {
  model: string;                    // Default: 'claude-opus-4-5'
  permissionMode: PermissionMode;   // Default: 'default'
  maxTokens: number;                // Default: 8192
  enableWebTools: boolean;
  enableVoice: boolean;
  enableVim: boolean;
  mcpServers: MCPServerConfig[];
  memoryDir: string;                // Default: ~/.claude/memory
  apiKey: string;                   // From ANTHROPIC_API_KEY env var
}

// Config is stored at ~/.claude/config.json
// Editable via /config slash command or directly
```

```bash
# Environment variables used by the system
export ANTHROPIC_API_KEY="..."          # Required
export CLAUDE_MODEL="claude-opus-4-5"  # Optional override
export CLAUDE_PERMISSION_MODE="bypass" # For CI: bypass | strict | default
export CLAUDE_MCP_SERVER_URL="..."     # Optional MCP server
export CLAUDE_MEMORY_DIR="~/.claude/memory"
```

---

## Common Patterns

### Pattern 1: Adding a New Tool

```typescript
// 1. Create src/tools/MyNewTool.ts
import { buildTool } from '../Tool';
import { z } from 'zod';

export const MyNewTool = buildTool({
  name: 'my_new_tool',
  description: 'What this tool does',
  inputSchema: z.object({
    input: z.string(),
  }),
  async execute({ input }, ctx) {
    const result = await doSomething(input);
    return { type: 'text', text: result };
  },
});

// 2. Register in src/tools.ts
import { MyNewTool } from './tools/MyNewTool';
export function getTools(config) {
  return [...existingTools, MyNewTool];
}
```

### Pattern 2: Adding a Slash Command

```typescript
// src/commands/myCommand.ts
export const myCommand = {
  name: 'my-command',
  description: 'Does something useful',
  usage: '/my-command [args]',
  async execute(args: string[], ctx: CommandContext) {
    const [target] = args;
    return ctx.query(`Please do something with: ${target}`);
  },
};

// Register in src/commands.ts
export const SLASH_COMMANDS = [...existingCommands, myCommand];
```

### Pattern 3: Reading Analysis Docs

```bash
# English architecture analysis
cat docs/en/architecture-overview.md

# Deep dive PDF
open docs/claude-code-deep-dive-xelatex.pdf
```

---

## Troubleshooting

### TypeScript source won't compile
```bash
# The source is decompiled — some types may need stubs
cd claude-code-source-code
ls stubs/   # Check existing stubs
# Add missing module stubs in stubs/ directory
```

### Python claw-code import errors
```bash
cd claw-code
pip install -e .   # Install in editable mode
python -c "from claw_code.agent import ClawAgent; print('OK')"
```

### Understanding a specific module
```bash
# Use grep to trace a concept through the codebase
grep -r "autoCompact" claude-code-source-code/src/ --include="*.ts" -l
grep -r "StreamingToolExecutor" claude-code-source-code/src/ --include="*.ts"
```

### Token/context issues in agent loop
```typescript
// Adjust compaction threshold in query.ts
const CONTEXT_COLLAPSE_THRESHOLD = 150_000; // tokens
const KEEP_LAST_N_MESSAGES = 20;
```

### Finding where a slash command is handled
```bash
grep -r "name: 'commit'" claude-code-source-code/src/commands/ --include="*.ts"
```
```
