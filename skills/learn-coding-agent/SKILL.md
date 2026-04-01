```markdown
---
name: learn-coding-agent
description: Architecture reference and deep analysis skill for Claude Code CLI agent internals, patterns, and implementation details
triggers:
  - "explain how claude code works internally"
  - "help me understand coding agent architecture"
  - "what is the claude code agent loop"
  - "how do claude code tools work"
  - "explain claude code permission system"
  - "how does claude code handle context compaction"
  - "what are claude code sub-agents"
  - "help me build a coding agent like claude code"
---

# Learn Coding Agent — Claude Code Architecture Reference

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection

A research repository documenting the internal architecture of Claude Code (v2.1.88), the highly popular CLI coding agent by Anthropic. This skill gives AI coding agents deep knowledge of agent loop patterns, tool systems, permission flows, telemetry, and production harness mechanisms derived from publicly available sources.

---

## What This Project Covers

This repository provides:
- **Deep analysis reports** on telemetry, hidden features, remote control, and future roadmap (EN/JA/KO/ZH)
- **Architecture diagrams** of the agent loop, tool system, and service layer
- **Code structure reference** for Claude Code's ~512K-line TypeScript codebase
- **12 progressive harness mechanisms** layered on top of the minimal agent loop
- Research for developers building their own coding agents

---

## The Minimal Agent Loop

The core pattern underlying Claude Code (and most coding agents):

```typescript
// Minimal agent loop — the foundation everything else builds on
async function agentLoop(userMessage: string) {
  const messages: Message[] = [
    { role: "user", content: userMessage }
  ];

  while (true) {
    const response = await claude.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 8096,
      tools: TOOL_DEFINITIONS,
      messages,
    });

    if (response.stop_reason === "end_turn") {
      // Final text response — return to user
      return extractText(response.content);
    }

    if (response.stop_reason === "tool_use") {
      // Execute each tool the model requested
      const toolResults = await executeTools(response.content);

      // Append assistant response + tool results and loop
      messages.push({ role: "assistant", content: response.content });
      messages.push({ role: "user", content: toolResults });
      // → loop back to API call
    }
  }
}
```

**Key insight**: This 20-line loop is the entire agent. Claude Code wraps it with a production harness of 12 mechanisms.

---

## The 12 Progressive Harness Mechanisms

Claude Code layers these on top of the minimal loop:

| # | Mechanism | Purpose |
|---|-----------|---------|
| 1 | **Streaming** | Stream tokens to terminal as they arrive |
| 2 | **Permission gate** | Block dangerous tools until user approves |
| 3 | **Parallel tool execution** | Run independent tools concurrently |
| 4 | **Context compaction** | Summarize old messages when near token limit |
| 5 | **Sub-agents** | Spawn child agent processes for subtasks |
| 6 | **Persistence** | Save/resume sessions across restarts |
| 7 | **MCP integration** | Connect external tool servers via protocol |
| 8 | **Cost tracking** | Accumulate and display API spend |
| 9 | **Retry + error handling** | Categorize and retry transient failures |
| 10 | **Telemetry** | Track usage events to Anthropic + Datadog |
| 11 | **Settings sync** | Remote-controlled feature flags via GrowthBook |
| 12 | **KAIROS mode** | Autonomous `<tick>` heartbeat for background tasks |

---

## Tool System Architecture

### Tool Interface Pattern

```typescript
// Based on Claude Code's Tool.ts pattern
interface Tool<TInput, TOutput> {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  
  // Permission check before execution
  checkPermission(input: TInput, context: PermissionContext): PermissionResult;
  
  // Actual execution
  execute(input: TInput, context: ExecutionContext): Promise<TOutput>;
  
  // Whether this tool can run in parallel with others
  isReadOnly: boolean;
}

// buildTool factory pattern
function buildTool<TInput, TOutput>(config: ToolConfig<TInput, TOutput>): Tool<TInput, TOutput> {
  return {
    name: config.name,
    description: config.description,
    inputSchema: config.inputSchema,
    checkPermission: config.checkPermission ?? defaultPermissionCheck,
    execute: config.execute,
    isReadOnly: config.isReadOnly ?? false,
  };
}

// Example: a simple file-reading tool
const readFileTool = buildTool({
  name: "read_file",
  description: "Read the contents of a file",
  isReadOnly: true,
  inputSchema: {
    type: "object",
    properties: {
      path: { type: "string", description: "Absolute or relative file path" }
    },
    required: ["path"]
  },
  checkPermission: (input, ctx) => {
    if (isPathOutsideWorkdir(input.path, ctx.workdir)) {
      return { allowed: false, reason: "Path outside working directory" };
    }
    return { allowed: true };
  },
  execute: async (input) => {
    const content = await fs.readFile(input.path, "utf-8");
    return { content, lines: content.split("\n").length };
  }
});
```

### Permission Flow

```typescript
// Permission decision hierarchy (Claude Code pattern)
type PermissionLevel = 
  | "always_allow"    // Pre-approved (e.g., read-only tools)
  | "ask_once"        // Prompt once, remember for session  
  | "ask_every_time"  // Prompt on each invocation
  | "never_allow";    // Blocked (e.g., rm -rf patterns)

async function executeWithPermission(
  tool: Tool,
  input: unknown,
  permissionStore: PermissionStore
): Promise<ToolResult> {
  const cached = permissionStore.get(tool.name, input);
  
  if (cached === "denied") {
    return { error: "Permission denied by user" };
  }
  
  if (cached !== "granted") {
    // Show permission dialog to user
    const decision = await promptUserForPermission(tool, input);
    permissionStore.set(tool.name, input, decision);
    
    if (decision === "denied") {
      return { error: "Permission denied by user" };
    }
  }
  
  return tool.execute(input);
}
```

---

## Parallel Tool Execution (StreamingToolExecutor Pattern)

```typescript
// Claude Code runs independent tools concurrently
async function executeToolsBatch(
  toolUseBlocks: ToolUseBlock[],
  tools: Map<string, Tool>
): Promise<ToolResult[]> {
  // Separate read-only (parallelizable) from write (sequential)
  const readOnly = toolUseBlocks.filter(t => tools.get(t.name)?.isReadOnly);
  const writes = toolUseBlocks.filter(t => !tools.get(t.name)?.isReadOnly);

  // Run all reads in parallel
  const readResults = await Promise.all(
    readOnly.map(block => executeSingleTool(block, tools))
  );

  // Run writes sequentially to avoid conflicts
  const writeResults: ToolResult[] = [];
  for (const block of writes) {
    writeResults.push(await executeSingleTool(block, tools));
  }

  // Reassemble in original order
  return toolUseBlocks.map(block => 
    [...readResults, ...writeResults].find(r => r.toolUseId === block.id)!
  );
}
```

---

## Context Compaction Pattern

```typescript
// When approaching token limit, summarize conversation history
async function compactContextIfNeeded(
  messages: Message[],
  currentTokenCount: number,
  tokenLimit: number
): Promise<Message[]> {
  const COMPACTION_THRESHOLD = 0.85; // compact at 85% of limit
  
  if (currentTokenCount < tokenLimit * COMPACTION_THRESHOLD) {
    return messages; // No compaction needed
  }

  // Keep system prompt + recent messages, summarize the middle
  const [systemMsg, ...rest] = messages;
  const recentMessages = rest.slice(-10); // Keep last 10 exchanges
  const oldMessages = rest.slice(0, -10);

  if (oldMessages.length === 0) return messages;

  // Ask Claude to summarize old context
  const summary = await claude.messages.create({
    model: "claude-haiku-4-5", // Use faster/cheaper model for compaction
    max_tokens: 2048,
    messages: [{
      role: "user",
      content: `Summarize the key facts, decisions, and code changes from this conversation for context continuity:\n\n${serializeMessages(oldMessages)}`
    }]
  });

  const summaryMessage: Message = {
    role: "user",
    content: `[Previous context summary]: ${extractText(summary.content)}`
  };

  return [systemMsg, summaryMessage, ...recentMessages];
}
```

---

## Sub-Agent Pattern

```typescript
// Claude Code spawns sub-agents for parallel subtasks
interface SubAgentTask {
  id: string;
  prompt: string;
  workdir: string;
  inheritedTools: string[]; // Which tools the sub-agent can use
}

async function spawnSubAgent(task: SubAgentTask): Promise<string> {
  // Sub-agents run in isolated contexts
  const subAgent = new AgentLoop({
    workdir: task.workdir,
    tools: filterTools(ALL_TOOLS, task.inheritedTools),
    maxTurns: 20,
    // Sub-agents report back via structured output
    systemPrompt: `You are a sub-agent. Complete the task and return a structured result.
Task ID: ${task.id}
Working directory: ${task.workdir}`
  });

  const result = await subAgent.run(task.prompt);
  return result;
}

// Orchestrator spawns multiple sub-agents
async function parallelSubAgents(tasks: SubAgentTask[]): Promise<string[]> {
  return Promise.all(tasks.map(spawnSubAgent));
}
```

---

## MCP (Model Context Protocol) Integration

```typescript
// Claude Code connects to external tool servers via MCP
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function connectMCPServer(config: MCPServerConfig): Promise<MCPConnection> {
  const transport = new StdioClientTransport({
    command: config.command,   // e.g., "npx"
    args: config.args,         // e.g., ["-y", "@modelcontextprotocol/server-filesystem"]
    env: { ...process.env, ...config.env }
  });

  const client = new Client(
    { name: "claude-code", version: "2.1.88" },
    { capabilities: { tools: {}, resources: {}, prompts: {} } }
  );

  await client.connect(transport);
  
  // Discover tools from this MCP server
  const { tools } = await client.listTools();
  
  return { client, tools, config };
}

// MCP config in ~/.claude/settings.json
const mcpConfig = {
  mcpServers: {
    filesystem: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
    },
    github: {
      command: "npx", 
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN }
    }
  }
};
```

---

## Cost Tracking Pattern

```typescript
// Accumulate costs across the session (from cost-tracker.ts pattern)
class CostTracker {
  private inputTokens = 0;
  private outputTokens = 0;
  private cacheReadTokens = 0;
  private cacheWriteTokens = 0;

  // Pricing per million tokens (approximate)
  private readonly PRICING = {
    "claude-opus-4-5":    { input: 15.00, output: 75.00, cacheRead: 1.50,  cacheWrite: 18.75 },
    "claude-sonnet-4-5":  { input: 3.00,  output: 15.00, cacheRead: 0.30,  cacheWrite: 3.75  },
    "claude-haiku-4-5":   { input: 0.80,  output: 4.00,  cacheRead: 0.08,  cacheWrite: 1.00  },
  };

  track(usage: APIUsage, model: string) {
    this.inputTokens     += usage.input_tokens ?? 0;
    this.outputTokens    += usage.output_tokens ?? 0;
    this.cacheReadTokens += usage.cache_read_input_tokens ?? 0;
    this.cacheWriteTokens+= usage.cache_creation_input_tokens ?? 0;
  }

  getTotalCost(model: string): number {
    const p = this.PRICING[model] ?? this.PRICING["claude-sonnet-4-5"];
    return (
      (this.inputTokens      / 1_000_000) * p.input +
      (this.outputTokens     / 1_000_000) * p.output +
      (this.cacheReadTokens  / 1_000_000) * p.cacheRead +
      (this.cacheWriteTokens / 1_000_000) * p.cacheWrite
    );
  }

  getSummary() {
    return {
      inputTokens: this.inputTokens,
      outputTokens: this.outputTokens,
      totalCost: this.getTotalCost("claude-sonnet-4-5")
    };
  }
}
```

---

## Retry & Error Handling Pattern

```typescript
// Claude Code's withRetry.ts pattern — categorize and retry
type ErrorCategory = 
  | "overloaded"      // 529 — retry with backoff
  | "rate_limited"    // 429 — wait for reset
  | "context_length"  // 400 context exceeded — compact and retry
  | "auth"            // 401 — don't retry
  | "fatal";          // Other — don't retry

function categorizeError(error: unknown): ErrorCategory {
  if (error instanceof APIError) {
    if (error.status === 529) return "overloaded";
    if (error.status === 429) return "rate_limited";
    if (error.status === 400 && error.message.includes("context")) return "context_length";
    if (error.status === 401) return "auth";
  }
  return "fatal";
}

async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const category = categorizeError(error);
      
      if (category === "auth" || category === "fatal") throw error;
      if (attempt === maxAttempts) throw error;

      const delay = category === "rate_limited" 
        ? 60_000                          // Wait 1 min for rate limit
        : Math.min(1000 * 2 ** attempt, 30_000); // Exponential backoff

      console.error(`Attempt ${attempt} failed (${category}), retrying in ${delay}ms...`);
      await sleep(delay);
    }
  }
  throw new Error("Unreachable");
}
```

---

## KAIROS Autonomous Mode Pattern

```typescript
// Future: KAIROS = fully autonomous agent with heartbeat ticks
// Based on research findings in docs/en/05-future-roadmap.md

interface KAIROSConfig {
  tickIntervalMs: number;        // Heartbeat interval
  pushNotificationWebhook: string;
  prSubscriptions: string[];     // GitHub PRs to monitor
  maxAutonomousActions: number;
}

// The <tick> mechanism — agent wakes up periodically
async function kairosDaemon(config: KAIROSConfig) {
  while (true) {
    // Agent receives a synthetic tick message
    const tickMessage = {
      role: "user" as const,
      content: `<tick timestamp="${new Date().toISOString()}">
Check your subscribed PRs, pending tasks, and notifications.
Take any appropriate autonomous actions.
</tick>`
    };

    await agentLoop(tickMessage.content);
    await sleep(config.tickIntervalMs);
  }
}
```

---

## Telemetry & Privacy Reference

From `docs/en/01-telemetry-and-privacy.md`:

```typescript
// What Claude Code collects on every event (simplified)
interface TelemetryEvent {
  // Identity
  userId: string;           // Hashed user ID
  sessionId: string;        // Session UUID
  
  // Environment fingerprint (collected on every event)
  osType: string;           // "darwin" | "linux" | "win32"
  nodeVersion: string;      
  repoHash: string;         // SHA of git remote URL — identifies your repo
  
  // Usage
  eventName: string;        // e.g., "tool_use", "message_sent"
  model: string;
  tokenCount: number;
  costUsd: number;
  
  // Two sinks
  // 1. Anthropic 1st-party (no UI opt-out)
  // 2. Datadog (can be disabled via env var)
}

// Enable full tool input capture (for debugging):
// OTEL_LOG_TOOL_DETAILS=1 claude

// Disable Datadog sink:
// ANTHROPIC_TELEMETRY_DISABLED=1 claude
```

---

## Settings & Feature Flags

```jsonc
// ~/.claude/settings.json — local configuration
{
  "model": "claude-sonnet-4-5",
  "theme": "dark",
  "autoApproveTools": ["read_file", "list_directory"],
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["./my-mcp-server.js"]
    }
  }
}
```

```bash
# Remote settings are polled hourly from Anthropic
# /api/claude_code/settings — can override any local setting
# Feature flags use GrowthBook with obfuscated names like:
#   tengu_frond_boric  →  enables Tengu model variant
#   capybara_v8_fast   →  enables fast mode for Capybara

# Internal Anthropic employees get additional capabilities:
# - Verification agents
# - Better system prompts  
# - Effort anchors
# - Undercover mode (auto-enabled in public repos)
```

---

## Repository Structure for Research

```
learn-coding-agent/
├── README.md              # Architecture overview + agent pattern
├── README_CN.md           # Chinese translation
├── README_KR.md           # Korean translation  
├── README_JA.md           # Japanese translation
└── docs/
    ├── en/                # English deep analysis reports
    │   ├── 01-telemetry-and-privacy.md
    │   ├── 02-hidden-features-and-codenames.md
    │   ├── 03-undercover-mode.md
    │   ├── 04-remote-control-and-killswitches.md
    │   └── 05-future-roadmap.md
    ├── ja/                # Japanese
    ├── ko/                # Korean
    └── zh/                # Chinese
```

---

## Quick Reference: Claude Code Stats

| Metric | Value |
|--------|-------|
| TypeScript files | ~1,884 |
| Total lines | ~512,664 |
| Largest file | `query.ts` (~785KB) — the main agent loop |
| Built-in tools | ~40+ |
| Slash commands | ~80+ |
| npm dependencies | ~192 packages |
| Runtime | Bun → compiled Node.js ≥18 bundle |

---

## Key Codenames Reference

| Codename | Meaning |
|----------|---------|
| **Capybara** | Claude model v8 (current) |
| **Tengu** | Internal model variant |
| **Fennec** | Maps to Opus 4.6 |
| **Numbat** | Next major model (unreleased) |
| **KAIROS** | Autonomous agent mode with heartbeat |

---

## Troubleshooting & Common Patterns

**Agent gets stuck in tool loop:**
```typescript
// Always enforce a max-turns limit
const MAX_TURNS = 50;
let turns = 0;

while (stopReason === "tool_use") {
  if (++turns > MAX_TURNS) {
    throw new Error(`Agent exceeded ${MAX_TURNS} turns — possible loop detected`);
  }
  // ... execute tools and loop
}
```

**Context window exhaustion:**
```typescript
// Monitor token usage and compact proactively
const usage = response.usage;
const utilizationPct = usage.input_tokens / MODEL_CONTEXT_WINDOW;

if (utilizationPct > 0.80) {
  messages = await compactContextIfNeeded(messages, usage.input_tokens, MODEL_CONTEXT_WINDOW);
}
```

**Parallel tool conflicts:**
```typescript
// Never run write tools in parallel — they may conflict
// Always check isReadOnly before parallelizing
const canParallelize = tools.every(t => toolMap.get(t.name)?.isReadOnly === true);
const results = canParallelize 
  ? await Promise.all(tools.map(execute))
  : await sequentialExecute(tools);
```

**Permission dialog on every run:**
```typescript
// Persist permissions across sessions in settings file
const sessionPermissions = await loadPermissions("~/.claude/permissions.json");
// Only prompt if not already in approved list
```
```
