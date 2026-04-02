```markdown
---
name: claude-code-book-agent-harness
description: Deep architectural guide for building AI Agent Harnesses based on Claude Code's design patterns — covers conversation loops, tool systems, permission pipelines, context compression, memory, hooks, sub-agents, and MCP integration.
triggers:
  - how does Claude Code work internally
  - build an agent harness from scratch
  - implement a conversation loop for an AI agent
  - tool permission pipeline design
  - context window management for agents
  - sub-agent fork pattern implementation
  - MCP protocol integration
  - agent memory system design
---

# Claude Code Book — Agent Harness Architecture

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A 420,000-character Chinese-language deep-dive into the architecture of Claude Code (Anthropic's AI coding agent), distilling its design into **transferable patterns** for building any production-grade Agent Harness. 15 chapters + 4 appendices, 139 architecture diagrams.

**Online reading:** https://lintsinghua.github.io

---

## What This Book Covers

The book reverse-engineers Claude Code's public behavior into concrete engineering patterns:

| Layer | Topics |
|---|---|
| **Foundation** | Async generator conversation loop, tool system, permission pipeline |
| **Core Systems** | Config/settings, memory, context compression, hook lifecycle |
| **Advanced Patterns** | Sub-agents, coordinator/worker, skill plugins, MCP integration |
| **Engineering** | Streaming architecture, Plan mode, building your own Harness |

---

## Reading the Book

### Online
```
https://lintsinghua.github.io
```

### Local Clone
```bash
git clone https://github.com/lintsinghua/claude-code-book.git
cd claude-code-book
```

Navigate chapters directly:
```
第一部分-基础篇/01-智能体编程的新范式.md
第一部分-基础篇/02-对话循环-Agent的心跳.md
第一部分-基础篇/03-工具系统-Agent的双手.md
第一部分-基础篇/04-权限管线-Agent的护栏.md
第二部分-核心系统篇/05-设置与配置-Agent的基因.md
第二部分-核心系统篇/06-记忆系统-Agent的长期记忆.md
第二部分-核心系统篇/07-上下文管理-Agent的工作记忆.md
第二部分-核心系统篇/08-钩子系统-Agent的生命周期扩展点.md
第三部分-高级模式篇/09-子智能体与Fork模式.md
第三部分-高级模式篇/10-协调器模式-多智能体编排.md
第三部分-高级模式篇/11-技能系统与插件架构.md
第三部分-高级模式篇/12-MCP集成与外部协议.md
第四部分-工程实践篇/13-流式架构与性能优化.md
第四部分-工程实践篇/14-Plan模式与结构化工作流.md
第四部分-工程实践篇/15-构建你自己的Agent-Harness.md
附录/A-源码导航地图.md
附录/B-工具完整清单.md
附录/C-功能标志速查表.md
附录/D-术语表.md
```

---

## Core Pattern 1: The Conversation Loop (Ch. 2)

The heartbeat of any Agent Harness is an `async generator` loop — not callbacks, not Promises:

```typescript
// Core Agent Harness conversation loop pattern
async function* agentLoop(
  initialMessages: Message[],
  deps: QueryDeps
): AsyncGenerator<AgentEvent> {
  const messages = [...initialMessages];

  while (true) {
    // 1. Pre-process: inject system context, memory, tool definitions
    const prepared = await prepareContext(messages, deps);

    // 2. Call LLM API with streaming
    yield { type: 'thinking' };
    const stream = await deps.llmClient.stream(prepared);

    // 3. Collect streamed response
    let assistantMessage = '';
    for await (const chunk of stream) {
      assistantMessage += chunk.text;
      yield { type: 'text_delta', delta: chunk.text };
    }

    messages.push({ role: 'assistant', content: assistantMessage });

    // 4. Parse tool calls from response
    const toolCalls = parseToolCalls(assistantMessage);

    if (toolCalls.length === 0) {
      // No tools needed — task complete
      yield { type: 'done', messages };
      return;
    }

    // 5. Execute tools and backfill results
    const toolResults = await executeTools(toolCalls, deps);
    for (const result of toolResults) {
      yield { type: 'tool_result', result };
      messages.push({ role: 'tool', content: result });
    }

    // 6. Check termination conditions
    const termination = checkTermination(messages, deps);
    if (termination.shouldStop) {
      yield { type: 'stopped', reason: termination.reason };
      return;
    }
    // Loop continues...
  }
}

// Usage
const agent = agentLoop(userMessages, deps);
for await (const event of agent) {
  switch (event.type) {
    case 'text_delta': process.stdout.write(event.delta); break;
    case 'tool_result': console.log('Tool:', event.result); break;
    case 'done': console.log('Complete'); break;
  }
}
```

**Why async generator?** Allows pausing at each yield point — tool execution, user confirmation, streaming chunks — without callback hell or Promise chaining complexity.

---

## Core Pattern 2: The Tool System (Ch. 3)

Every tool follows a 5-element protocol:

```typescript
interface Tool<TInput, TOutput, TProgress = never> {
  name: string;                          // Unique identifier
  inputSchema: ZodSchema<TInput>;        // Validated input (Zod v4)
  permissions: ToolPermissions;          // readOnly, destructive, concurrencySafe
  execute: (
    input: TInput,
    context: ToolContext
  ) => AsyncGenerator<TProgress | TOutput>;
  renderResult: (output: TOutput) => React.ReactNode; // Terminal UI
}

// Tool factory with fail-safe defaults
function buildTool<TInput, TOutput>(
  definition: ToolDefinition<TInput, TOutput>
): Tool<TInput, TOutput> {
  return {
    ...definition,
    execute: async function* (input, context) {
      // Validate input against schema
      const parsed = definition.inputSchema.safeParse(input);
      if (!parsed.success) {
        yield { type: 'error', message: parsed.error.message };
        return;
      }
      yield* definition.execute(parsed.data, context);
    }
  };
}

// Example: read-only file tool
const readFileTool = buildTool({
  name: 'read_file',
  inputSchema: z.object({
    path: z.string(),
    encoding: z.enum(['utf8', 'base64']).default('utf8'),
  }),
  permissions: { readOnly: true, destructive: false, concurrencySafe: true },
  async *execute({ path, encoding }, { workDir }) {
    const fullPath = resolve(workDir, path);
    const content = await fs.readFile(fullPath, encoding);
    yield { type: 'success', content };
  },
  renderResult: ({ content }) => <Text>{content}</Text>,
});
```

### Concurrent Tool Execution

```typescript
// Safe tools run in parallel; unsafe tools run exclusively
async function executeTools(
  toolCalls: ToolCall[],
  registry: ToolRegistry
): Promise<ToolResult[]> {
  const partitions = partitionByConcurrency(toolCalls, registry);

  const results: ToolResult[] = [];
  for (const partition of partitions) {
    if (partition.type === 'parallel') {
      // Safe tools: greedy parallel execution
      const batch = await Promise.all(
        partition.calls.map(call => executeSingle(call, registry))
      );
      results.push(...batch);
    } else {
      // Non-safe tools: sequential, exclusive
      for (const call of partition.calls) {
        results.push(await executeSingle(call, registry));
      }
    }
  }
  return results;
}
```

---

## Core Pattern 3: The Permission Pipeline (Ch. 4)

Four-stage fail-closed pipeline — all stages must pass:

```typescript
async function checkPermission(
  toolCall: ToolCall,
  context: PermissionContext
): Promise<PermissionResult> {

  // Stage 1: Schema validation (always first)
  const schemaResult = validateSchema(toolCall);
  if (!schemaResult.ok) return { allowed: false, reason: 'schema_invalid' };

  // Stage 2: Rule matching (bash allow/deny lists, path globs)
  const ruleResult = matchRules(toolCall, context.rules);
  if (ruleResult.explicit === 'deny') return { allowed: false, reason: 'rule_denied' };
  if (ruleResult.explicit === 'allow') return { allowed: true, reason: 'rule_allowed' };

  // Stage 3: Context evaluation (mode, trust level, risk score)
  const contextResult = evaluateContext(toolCall, context);
  if (contextResult.autoApprove) return { allowed: true, reason: 'context_auto' };

  // Stage 4: Interactive confirmation (with speculative classifier)
  return await requestConfirmation(toolCall, context);
}

// Speculative classifier: race the user prompt against a fast classifier
async function requestConfirmation(
  toolCall: ToolCall,
  context: PermissionContext
): Promise<PermissionResult> {
  const classifierPromise = speculativeClassify(toolCall); // ~2s fast model
  const userPromise = promptUser(toolCall);                 // waits for input

  // If classifier finishes first and is confident, skip user prompt
  const winner = await Promise.race([
    classifierPromise.then(r => ({ source: 'classifier', result: r })),
    userPromise.then(r => ({ source: 'user', result: r })),
  ]);

  return winner.result;
}

// Permission modes (least → most permissive)
type PermissionMode =
  | 'default'   // Interactive confirmation for all destructive ops
  | 'plan'      // Read-only; write ops blocked
  | 'auto'      // Auto-approve based on rules
  | 'bubble'    // Escalate to parent agent
  | 'bypass';   // Trust all (CI/CD use only)
```

---

## Core Pattern 4: Context Compression (Ch. 7)

Four-level progressive compression when approaching token limits:

```typescript
// Effective window = total_context - reserved_output - safety_buffer
const EFFECTIVE_WINDOW = 200_000 - 32_000 - 8_000; // = 160,000 tokens

async function manageContext(
  messages: Message[],
  tokenCount: number
): Promise<Message[]> {

  if (tokenCount < EFFECTIVE_WINDOW * 0.6) return messages; // No action needed

  if (tokenCount < EFFECTIVE_WINDOW * 0.75) {
    // Level 1: Snip — truncate oldest non-essential messages
    return snipOldMessages(messages, { keepSystemPrompt: true, keepRecent: 20 });
  }

  if (tokenCount < EFFECTIVE_WINDOW * 0.85) {
    // Level 2: MicroCompact — summarize tool result bodies
    return microCompactToolResults(messages);
  }

  if (tokenCount < EFFECTIVE_WINDOW * 0.95) {
    // Level 3: Collapse — merge consecutive same-role messages
    return collapseMessages(messages);
  }

  // Level 4: AutoCompact — full LLM-based summarization
  return await autoCompact(messages);
}

// AutoCompact uses two-phase prompting: analysis (discarded) + summary (kept)
async function autoCompact(messages: Message[]): Promise<Message[]> {
  const compressionPrompt = `
Analyze the conversation history and produce a structured summary.

<analysis>
[Your working analysis — this section will be DISCARDED]
</analysis>

<summary>
## Completed Work
[What has been accomplished]

## Current State  
[File contents, decisions made, open questions]

## Next Steps
[What remains to do]
</summary>`;

  const compressed = await llm.complete(compressionPrompt + formatMessages(messages));
  // Extract only the <summary> block
  const summary = extractSummary(compressed);

  return [
    { role: 'system', content: 'Previous conversation compressed:' },
    { role: 'assistant', content: summary },
  ];
}
```

**Circuit breaker:** After 3 consecutive compression failures, halt and surface error to user rather than looping.

---

## Core Pattern 5: Fork / Sub-Agent (Ch. 9)

Sub-agents inherit parent context via byte-level copy (maximizing prompt cache hits):

```typescript
interface ForkOptions {
  agentType: 'explore' | 'plan' | 'general' | 'verification';
  inheritContext: boolean;       // Copy parent's CacheSafeParams
  maxDepth: number;              // Prevent recursive fork explosion
  isolatedTools?: string[];      // Restrict available tools
}

async function forkSubAgent(
  parentContext: AgentContext,
  task: string,
  options: ForkOptions
): Promise<AgentResult> {

  // Guard: prevent recursive fork explosion
  if (parentContext.forkDepth >= options.maxDepth) {
    throw new Error(`Max fork depth ${options.maxDepth} exceeded`);
  }

  // Inherit cache-safe params (system prompt, memory, tool defs — stable content)
  const childContext: AgentContext = {
    ...parentContext.cacheSafeParams,  // Maximizes cache hit area
    forkDepth: parentContext.forkDepth + 1,
    task,
    tools: options.isolatedTools
      ? filterTools(parentContext.tools, options.isolatedTools)
      : parentContext.tools,
    // Use placeholder for parent's last tool result (cache-friendly)
    parentResultPlaceholder: CACHE_PLACEHOLDER,
  };

  // Run sub-agent to completion
  const subAgent = agentLoop(
    [{ role: 'user', content: task }],
    buildDepsForFork(childContext)
  );

  const results: AgentEvent[] = [];
  for await (const event of subAgent) {
    results.push(event);
  }

  return extractResult(results);
}

// Built-in agent types and their tool restrictions
const AGENT_CONFIGS = {
  explore:       { readOnly: true,  tools: ['read_file', 'search', 'list_dir'] },
  plan:          { readOnly: true,  tools: ['read_file', 'search', 'write_plan'] },
  general:       { readOnly: false, tools: 'all' },
  verification:  { readOnly: true,  tools: ['read_file', 'run_tests', 'lint'] },
};
```

---

## Core Pattern 6: MCP Integration (Ch. 12)

```typescript
// 8 supported transport protocols
type MCPTransport =
  | { type: 'stdio'; command: string; args: string[] }
  | { type: 'sse'; url: string }
  | { type: 'http'; url: string }
  | { type: 'ws'; url: string }
  | { type: 'sdk'; module: string };

// Tool naming: mcp__{server}__{tool}
const MCP_TOOL_PREFIX = (server: string, tool: string) =>
  `mcp__${server}__${tool}`;

// Connection manager with 5-state lifecycle
type MCPConnectionState =
  | 'disconnected'
  | 'connecting'
  | 'connected'
  | 'error'
  | 'disabled';

class MCPConnectionManager {
  private connections = new Map<string, MCPConnection>();

  async connect(server: MCPServerConfig): Promise<void> {
    const conn = this.connections.get(server.name) ?? this.createConnection(server);
    this.connections.set(server.name, conn);
    await conn.initialize();
    // Register server's tools into the global tool registry
    const tools = await conn.listTools();
    tools.forEach(tool =>
      this.registry.register({
        name: MCP_TOOL_PREFIX(server.name, tool.name),
        ...adaptMCPTool(tool),
      })
    );
  }
}

// claude_desktop_config.json / .claude/settings.json MCP config
const mcpConfig = {
  mcpServers: {
    filesystem: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      type: "stdio"
    },
    github: {
      url: "https://api.githubcopilot.com/mcp/",
      type: "http",
      headers: { Authorization: `Bearer ${process.env.GITHUB_TOKEN}` }
    }
  }
};
```

---

## Core Pattern 7: Hook System (Ch. 8)

26 lifecycle events across 5 hook types:

```typescript
// Hook response protocol
interface HookResponse {
  action: 'approve' | 'block' | 'modify';
  updatedInput?: unknown;        // Modified tool input
  additionalContext?: string;    // Injected into next LLM prompt
  reason?: string;               // Shown to user on block
}

// SKILL.md / config hook registration
const hookConfig = {
  hooks: {
    // Intercept before any tool call
    'tool:before': [
      {
        type: 'command',
        command: 'python3 audit_tool.py',
        timeout: 5000,
      }
    ],
    // Post-process bash output
    'tool:after:bash': [
      {
        type: 'function',
        handler: async (event) => {
          if (event.output.includes('SECRET')) {
            return { action: 'block', reason: 'Secret detected in output' };
          }
          return { action: 'approve' };
        }
      }
    ],
    // Inject context before LLM call
    'prompt:before': [
      {
        type: 'http',
        url: `${process.env.CONTEXT_SERVICE_URL}/enrich`,
        method: 'POST',
      }
    ]
  }
};

// Hook execution with timeout and error isolation
async function executeHook(
  hook: HookConfig,
  event: HookEvent
): Promise<HookResponse> {
  const timeout = hook.timeout ?? 10_000;
  try {
    return await Promise.race([
      runHook(hook, event),
      sleep(timeout).then(() => ({ action: 'approve' as const })), // Fail open on timeout
    ]);
  } catch {
    return { action: 'approve' }; // Hooks never crash the agent
  }
}
```

---

## Core Pattern 8: Memory System (Ch. 6)

```typescript
// Four memory types — all write-once, append-friendly
type MemoryType = 'user' | 'feedback' | 'project' | 'reference';

// Memory design principle: only save what can't be derived from current state
interface MemoryEntry {
  type: MemoryType;
  content: string;
  timestamp: number;
  tags: string[];
}

// MEMORY.md index file limits: 200 lines / 25KB
const MEMORY_LIMITS = { maxLines: 200, maxBytes: 25 * 1024 };

// Fork memory extraction — auto-extracted, exclusive to sub-agent
async function extractForkMemory(
  parentMessages: Message[],
  task: string
): Promise<MemoryEntry[]> {
  // Sub-agent gets relevant memory slice; parent's memory writer is paused
  const relevant = await semanticSearch(
    parentMessages,
    task,
    { topK: 10, threshold: 0.7 }
  );
  return relevant.map(adaptToMemoryEntry);
}

// CacheSafeParams: memory must be stable across turns for cache sharing
interface CacheSafeParams {
  systemPrompt: string;      // Stable
  memorySnapshot: string;    // Stable snapshot — not live
  toolDefinitions: string;   // Stable JSON
  projectContext: string;    // Stable
  userPreferences: string;   // Stable
}
```

---

## Building Your Own Harness: 6-Step Roadmap (Ch. 15)

```
Step 1: AsyncGenerator conversation loop (Ch. 2 pattern)
   └─ Wire: LLM client → stream parser → event emitter

Step 2: Fail-closed tool system (Ch. 3 pattern)
   └─ Wire: Zod schema validation → tool registry → concurrent executor

Step 3: Four-phase permission pipeline (Ch. 4 pattern)
   └─ Wire: schema → rules → context → interactive confirmation

Step 4: Snip + Summary context management (Ch. 7 pattern)
   └─ Wire: token counter → compression threshold → compressor chain

Step 5: Memory storage (Ch. 6 pattern)
   └─ Wire: MEMORY.md reader/writer → cache-safe snapshot → fork isolation

Step 6: Hook executor (Ch. 8 pattern)
   └─ Wire: lifecycle event bus → hook runner → fail-open timeout
```

### Decision Matrix: Do You Need a Harness?

| Requirement | Simple API Call | Agent Harness |
|---|---|---|
| Multi-turn conversation | ❌ | ✅ |
| Tool execution | ❌ | ✅ |
| Context > 50K tokens | ❌ | ✅ |
| Permission control | ❌ | ✅ |
| Sub-agent delegation | ❌ | ✅ |
| Single Q&A | ✅ | Overkill |

---

## Configuration System (Ch. 5)

Six-layer config priority (highest wins):

```
plugin → user → project → local → feature-flag → policy
```

```typescript
// Merge rules by value type:
// - Arrays: concat + deduplicate  → ['a','b'] + ['b','c'] = ['a','b','c']
// - Objects: deep merge           → {x:1} + {y:2} = {x:1, y:2}
// - Scalars: higher layer wins    → 'foo' overrides 'bar'

// Security: projectSettings excluded from security checks
// (prevents malicious repo from hijacking agent permissions via .claude/settings.json)

// Feature flags: two-layer system
const isEnabled = (flag: string): boolean => {
  // Layer 1: compile-time (bundled flags, zero runtime cost)
  if (COMPILE_TIME_FLAGS[flag] !== undefined) return COMPILE_TIME_FLAGS[flag];
  // Layer 2: runtime (GrowthBook, A/B testing, gradual rollout)
  return growthBook.isOn(flag);
};
```

---

## Quick Reference: Appendices

| Appendix | Content | Use When |
|---|---|---|
| **A** — Architecture Map | 16 core modules, dependency tree, 6 data flow paths | Orienting in codebase |
| **B** — Tool Catalog | 50+ tools, 12 categories, readOnly/destructive/concurrencySafe flags | Choosing/implementing tools |
| **C** — Feature Flags | 89 flags, 13 categories, compile-time vs runtime | Configuring environments |
| **D** — Glossary | 100 terms, Chinese/English, cross-references | Terminology lookup |

---

## Key Architectural Insights

1. **Async generator > callbacks**: Allows natural pause/resume at every yield point — tool execution, user confirmation, streaming
2. **Fail-closed permissions**: All 4 pipeline stages must explicitly pass; any failure = deny
3. **Cache-aware design**: `CacheSafeParams` separates stable (cacheable) from dynamic (non-cacheable) context — critical for latency
4. **Circuit breaker for compression**: 3 consecutive failures → halt, surface to user (from 1,279 real sessions)
5. **Fork inherits bytes, not references**: Maximizes prompt cache hit area across parent/child agents
6. **Hooks never crash the agent**: Fail-open on timeout/error — hooks are advisory, not load-bearing

---

## Troubleshooting

**Context compression triggering too aggressively**
→ Check `EFFECTIVE_WINDOW` calculation; reserved output tokens are often underestimated for code-heavy tasks.

**Tool permissions always denying**
→ Pipeline is fail-closed by design. Check: (1) Zod schema matches actual input shape, (2) rule patterns use correct glob syntax, (3) mode is not `plan` (read-only).

**Sub-agent fork depth exceeded**
→ Set explicit `maxDepth` per task type. Verification agents should never fork. Use `explore` type (read-only) for research tasks.

**MCP server tools not appearing**
→ Tool names must match pattern `mcp__{server}__{tool}`. Check `MCPConnectionState` — server may be in `error` state silently.

**Memory growing beyond limits**
→ `MEMORY.md` caps at 200 lines / 25KB. Implement periodic compaction: summarize old entries, preserve only entries with `tags` matching active project context.

**Prompt cache misses on fork**
→ Ensure `CacheSafeParams` contains only stable content. Dynamic values (timestamps, request IDs, mutable file contents) must be excluded from the 5 cache-safe dimensions.
```
