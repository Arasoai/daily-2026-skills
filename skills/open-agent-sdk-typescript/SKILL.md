```markdown
---
name: open-agent-sdk-typescript
description: TypeScript SDK for running Claude-powered agent loops in-process without CLI dependencies
triggers:
  - use open agent sdk
  - run agent loop in typescript
  - claude agent without cli
  - open-agent-sdk setup
  - create ai agent typescript
  - agent sdk streaming query
  - mcp server integration typescript
  - subagent definition sdk
---

# Open Agent SDK (TypeScript)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Open Agent SDK runs the full agentic loop **in-process** — no subprocess, no CLI wrapper. It's a drop-in open-source alternative to `claude-agent-sdk` that works in cloud, serverless, Docker, and CI/CD environments.

## Installation

```bash
npm install @codeany/open-agent-sdk
```

**Required environment variable:**

```bash
export CODEANY_API_KEY=your-api-key
```

**Optional — use third-party providers (e.g. OpenRouter):**

```bash
export CODEANY_BASE_URL=https://openrouter.ai/api
export CODEANY_API_KEY=sk-or-...
export CODEANY_MODEL=anthropic/claude-sonnet-4
```

## Core Concepts

- **`query()`** — one-shot streaming generator; yields `SDKMessage` objects
- **`createAgent()`** — reusable agent with session persistence and multi-turn support
- **`tool()` / `defineTool()`** — register custom tools (Zod or raw JSON schema)
- **`createSdkMcpServer()`** — bundle tools as an in-process MCP server
- **`getAllBaseTools()`** — returns all 34 built-in file/shell tools

## Quick Start Patterns

### Streaming one-shot query

```typescript
import { query } from "@codeany/open-agent-sdk";

for await (const message of query({
  prompt: "Read package.json and tell me the project name.",
  options: {
    allowedTools: ["Read", "Glob"],
    permissionMode: "bypassPermissions",
  },
})) {
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if ("text" in block) console.log(block.text);
    }
  }
  if (message.type === "result") {
    console.log(`Cost: $${message.total_cost_usd?.toFixed(4)}`);
  }
}
```

### Blocking prompt with `createAgent`

```typescript
import { createAgent } from "@codeany/open-agent-sdk";

const agent = createAgent({ model: "claude-sonnet-4-6" });
const result = await agent.prompt("What files are in this project?");

console.log(result.text);
console.log(`Turns: ${result.num_turns}`);
console.log(`Tokens: ${result.usage.input_tokens + result.usage.output_tokens}`);
```

### Multi-turn conversation

```typescript
import { createAgent } from "@codeany/open-agent-sdk";

const agent = createAgent({ maxTurns: 5 });

const r1 = await agent.prompt('Create a file /tmp/hello.txt with "Hello World"');
console.log(r1.text);

const r2 = await agent.prompt("Read back the file you just created");
console.log(r2.text);

console.log(`Session messages: ${agent.getMessages().length}`);

await agent.close(); // persists session, closes MCP connections
```

## Custom Tools

### With Zod schema (recommended)

```typescript
import { z } from "zod";
import { query, tool, createSdkMcpServer } from "@codeany/open-agent-sdk";

const getWeather = tool(
  "get_weather",
  "Get the temperature for a city",
  { city: z.string().describe("City name") },
  async ({ city }) => ({
    content: [{ type: "text", text: `${city}: 22°C, sunny` }],
  }),
);

const server = createSdkMcpServer({ name: "weather", tools: [getWeather] });

for await (const msg of query({
  prompt: "What is the weather in Tokyo and London?",
  options: { mcpServers: { weather: server } },
})) {
  if (msg.type === "result") console.log(`Done: $${msg.total_cost_usd?.toFixed(4)}`);
}
```

### Low-level tool with `defineTool`

```typescript
import { createAgent, getAllBaseTools, defineTool } from "@codeany/open-agent-sdk";

const calculator = defineTool({
  name: "Calculator",
  description: "Evaluate a math expression",
  inputSchema: {
    type: "object",
    properties: { expression: { type: "string" } },
    required: ["expression"],
  },
  isReadOnly: true,
  async call(input) {
    const result = Function(`'use strict'; return (${input.expression})`)();
    return `${input.expression} = ${result}`;
  },
});

const agent = createAgent({ tools: [...getAllBaseTools(), calculator] });
const r = await agent.prompt("Calculate 2**10 * 3");
console.log(r.text);
```

## MCP Server Integration

```typescript
import { createAgent } from "@codeany/open-agent-sdk";

const agent = createAgent({
  mcpServers: {
    filesystem: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
    },
    github: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
    },
  },
});

const result = await agent.prompt("List files in /tmp");
console.log(result.text);
await agent.close();
```

## Subagents

```typescript
import { query } from "@codeany/open-agent-sdk";

for await (const msg of query({
  prompt: "Use the code-reviewer agent to review src/index.ts",
  options: {
    agents: {
      "code-reviewer": {
        description: "Expert code reviewer",
        prompt: "Analyze code quality. Focus on security and performance.",
        tools: ["Read", "Glob", "Grep"],
      },
    },
  },
})) {
  if (msg.type === "assistant") {
    for (const block of msg.message.content) {
      if ("text" in block) process.stdout.write(block.text);
    }
  }
}
```

## Permission Modes

| Mode                 | Behavior                                      |
|----------------------|-----------------------------------------------|
| `bypassPermissions`  | All tools allowed without prompting (default) |
| `acceptEdits`        | Auto-accept file edits, prompt for others     |
| `dontAsk`            | Silently skip disallowed tools                |
| `default`            | Prompt user for each sensitive operation      |
| `plan`               | Plan only, no writes                          |

```typescript
// Read-only analysis agent
for await (const msg of query({
  prompt: "Review src/ for security issues.",
  options: {
    allowedTools: ["Read", "Glob", "Grep"],
    permissionMode: "dontAsk",
  },
})) { /* ... */ }

// Custom permission callback
const agent = createAgent({
  canUseTool: async (tool, input) => {
    if (tool.name === "Write") return { granted: false, reason: "Read-only mode" };
    return { granted: true };
  },
});
```

## Session Management

```typescript
import {
  createAgent,
  listSessions,
  getSessionMessages,
  forkSession,
} from "@codeany/open-agent-sdk";

// Resume a previous session
const agent = createAgent({ resume: "session-id-here" });

// Continue most recent session
const agent2 = createAgent({ continue: true });

// List all persisted sessions
const sessions = await listSessions();

// Retrieve messages from a session
const messages = await getSessionMessages("session-id");

// Fork a session for branching experiments
const forkedId = await forkSession("session-id");

// Disable persistence
const ephemeral = createAgent({ persistSession: false });
```

## Structured Output

```typescript
import { createAgent } from "@codeany/open-agent-sdk";

const agent = createAgent({
  outputFormat: {
    type: "json_schema",
    schema: {
      type: "object",
      properties: {
        summary: { type: "string" },
        issues: { type: "array", items: { type: "string" } },
        severity: { type: "string", enum: ["low", "medium", "high"] },
      },
      required: ["summary", "issues", "severity"],
    },
  },
});

const result = await agent.prompt("Review src/auth.ts for security issues");
const parsed = JSON.parse(result.text);
console.log(parsed.severity, parsed.issues);
```

## Lifecycle Hooks

```typescript
import { createAgent } from "@codeany/open-agent-sdk";

const agent = createAgent({
  hooks: {
    preToolCall: [
      {
        matcher: { toolName: "Write" },
        callback: async (tool, input) => {
          console.log(`Writing to: ${input.path}`);
        },
      },
    ],
    postToolCall: [
      {
        matcher: { toolName: "*" },
        callback: async (tool, input, output) => {
          console.log(`Tool ${tool.name} completed`);
        },
      },
    ],
  },
});
```

## Agent Methods Reference

```typescript
const agent = createAgent({ maxTurns: 10, model: "claude-sonnet-4-6" });

// Streaming
for await (const msg of agent.query("Explain this codebase")) { /* ... */ }

// Blocking
const result = await agent.prompt("Write tests for utils.ts");

// Session control
agent.getMessages();           // full conversation history
agent.clear();                 // reset session
agent.interrupt();             // abort current query
agent.setModel("claude-opus-4");
agent.setPermissionMode("dontAsk");
await agent.close();           // persist + cleanup
```

## Key Options

```typescript
createAgent({
  model: "claude-sonnet-4-6",          // LLM model
  apiKey: process.env.CODEANY_API_KEY, // or set env var
  baseURL: process.env.CODEANY_BASE_URL,
  cwd: "/path/to/project",             // working directory
  systemPrompt: "You are a...",        // override system prompt
  appendSystemPrompt: "Always...",     // append to default
  maxTurns: 10,
  maxBudgetUsd: 0.50,                  // spending cap
  thinking: { type: "adaptive" },      // extended thinking
  effort: "high",                      // low | medium | high | max
  env: { MY_VAR: "value" },            // env vars for tools
  abortController: new AbortController(),
});
```

## Web UI (Testing)

```bash
npx tsx examples/web/server.ts
# Open http://localhost:8081
```

## SDKMessage Types

```typescript
// Message types yielded by query() and agent.query()
type SDKMessage =
  | { type: "assistant"; message: { content: ContentBlock[] } }
  | { type: "user"; message: { content: ContentBlock[] } }
  | { type: "result"; total_cost_usd?: number; usage: Usage }
  | { type: "system"; subtype: string; [key: string]: unknown };

// Check for text content
if (message.type === "assistant") {
  for (const block of message.message.content) {
    if ("text" in block) console.log(block.text);
    if (block.type === "tool_use") console.log(`Using: ${block.name}`);
  }
}
```

## Troubleshooting

**`Error: CODEANY_API_KEY is not set`**
→ Export the env var: `export CODEANY_API_KEY=your-key`

**Agent runs too many turns / hits limit**
→ Increase `maxTurns` or check if the task is resolvable with given `allowedTools`

**MCP server not connecting**
→ Ensure the MCP server binary is installed (`npx -y @modelcontextprotocol/server-*`), call `await agent.close()` to clean up

**Tools not available**
→ By default all 34 built-in tools are available; if you pass `tools: [...]` it replaces them. Use `[...getAllBaseTools(), myTool]` to extend

**Session not persisting**
→ Confirm `persistSession: true` (default) and that `agent.close()` is awaited

**Custom base URL with OpenRouter**
→ Set `CODEANY_BASE_URL=https://openrouter.ai/api` and `CODEANY_MODEL=anthropic/claude-sonnet-4` alongside your OpenRouter API key
```
