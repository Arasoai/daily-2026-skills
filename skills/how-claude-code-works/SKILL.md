```markdown
---
name: how-claude-code-works
description: Deep dive into Claude Code internals — architecture, agent loop, context engineering, tool system, and security for building or understanding AI coding agents.
triggers:
  - how does claude code work internally
  - explain claude code architecture
  - how does the claude code agent loop work
  - claude code context engineering and compression
  - claude code tool system internals
  - how does claude code handle security and permissions
  - build my own ai coding agent like claude code
  - claude code hooks and extensibility
---

# How Claude Code Works

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A 12-chapter deep dive into Claude Code's 500K+ line TypeScript source, covering architecture, the agent loop, context engineering, tool systems, permissions, multi-agent coordination, memory, skills, and UX design. Companion project to [`claude-code-from-scratch`](https://github.com/Windy3f3f3f3f/claude-code-from-scratch) — 1,300 lines, 8 chapters, build your own Claude Code.

## Reading the Docs

**Online:** https://windy3f3f3f3f.github.io/how-claude-code-works/#/

**Local:**
```bash
git clone https://github.com/Windy3f3f3f3f/how-claude-code-works
cd how-claude-code-works
# Open docs/ folder — plain Markdown, readable in any editor
```

**Chapter map:**

| File | Topic |
|------|-------|
| `docs/quick-start.md` | 10-minute condensed overview |
| `docs/01-overview.md` | Tech choices, 9-phase startup, data flow |
| `docs/02-agent-loop.md` | Dual-layer loop, 7 Continue Sites, streaming tool execution |
| `docs/03-context-engineering.md` | 4-level compression pipeline, cache strategy |
| `docs/04-tool-system.md` | 66 tools, MCP, concurrency, OAuth |
| `docs/05-code-editing-strategy.md` | search-and-replace, uniqueness constraint |
| `docs/06-hooks-extensibility.md` | 23+ hook events, 6-stage pipeline |
| `docs/07-multi-agent.md` | Sub-agents, coordinator, Swarm, Worktree isolation |
| `docs/08-memory-system.md` | 4 memory types, semantic recall, drift defense |
| `docs/09-skills-system.md` | 6-layer skill sources, lazy load, token budget |
| `docs/10-permission-security.md` | 5-layer defense, AST analysis, 23 safety checks |
| `docs/11-user-experience.md` | Ink renderer, Yoga Flexbox, virtual scroll |
| `docs/12-minimal-components.md` | Minimal viable agent, 500→500K line roadmap |

---

## Key Architecture Concepts

### System Architecture

```
User Input
    │
    ▼
QueryEngine (session management)
    │
    ▼
query() main loop ◄──────────────────────┐
    │                                     │
    ▼                                     │
Claude API (streaming)                    │
    │                                     │
    ├─► Text tokens → stream to terminal  │
    │                                     │
    └─► Tool calls → ToolExecutionEngine  │
              │                           │
              ├─ ReadFile                 │
              ├─ EditFile                 │
              ├─ Shell                    │
              ├─ Search                   │
              └─ MCP Tools               │
                      │                  │
                      └─ results ────────┘

Context Engineering layer feeds:
  - System prompt
  - Git status
  - CLAUDE.md files
  - Compression pipeline
```

### The Agent Loop (docs/02-agent-loop.md)

Claude Code uses a **dual-layer loop**:

1. **Outer loop** — manages conversation state, compaction triggers, session lifecycle
2. **Inner loop** — single API call → parse response → execute tools → inject results → repeat

**7 Continue Sites** (fault recovery strategies):
- `CONTINUE` — normal tool result injection
- `CONTINUE_WITH_COMPACTION` — context too long, compress then continue
- `CONTINUE_WITH_MAX_TOKENS_RETRY` — upgrade 4K→64K output limit and retry
- `STOP_WITH_RESULT` — final answer reached
- `STOP_WITH_ERROR` — unrecoverable error
- `STOP_WITH_INTERRUPT` — user cancelled
- `STOP_WITH_LIMIT` — turn/cost limit hit

**Tool pre-execution (StreamingToolExecutor):** While the model streams its response, the system parses tool calls and begins executing them concurrently. The ~1s tool I/O latency is hidden inside the model's 5–30s generation window.

### 4-Level Context Compression (docs/03-context-engineering.md)

When context approaches the limit, compression triggers progressively:

```
Level 1: TRUNCATE
  └─ Cut large tool outputs in older messages (fast, lossy for old data)

Level 2: DEDUPLICATE  
  └─ Remove repeated content (near-zero cost)

Level 3: FOLD
  └─ Collapse inactive conversation segments (reversible, content intact)

Level 4: SUMMARIZE
  └─ Launch sub-agent to summarize entire conversation (last resort)
```

After any compression, the system **auto-restores**:
- The 5 most recently edited files (full content re-injected)
- Active skill context (prevents the model forgetting what it was doing)

### Tool System (docs/04-tool-system.md)

All 66+ tools share one interface:

```typescript
interface Tool {
  name: string;
  description: string;
  inputSchema: ZodSchema;
  execute(input: unknown, context: ToolContext): Promise<ToolResult>;
  readonly: boolean;        // true = can run in parallel
  requiresPermission: boolean;
}
```

**Concurrency rules (automatic):**
- Read-only tools → parallel execution
- Write tools → serialized automatically
- Output > 100K chars → written to disk, model receives path + summary

### 5-Layer Permission System (docs/10-permission-security.md)

```
Layer 1: Permission Mode
  └─ Trust level restricts available operation classes

Layer 2: Rule Matching
  └─ Command pattern whitelist/blacklist

Layer 3: Bash AST Analysis (tree-sitter)
  └─ 23 safety checks on parsed shell AST, not regex:
     - Command injection detection
     - Env variable leak detection  
     - Special character attacks
     - Pipe chain analysis

Layer 4: User Confirmation
  └─ Dangerous ops require explicit confirm
     200ms debounce prevents accidental keypress confirmation

Layer 5: Hook Validation
  └─ User-defined rules, can mutate tool inputs
     (e.g. auto-add --dry-run to rm commands)
```

### Hooks System (docs/06-hooks-extensibility.md)

Configure in `.claude/hooks.json` or `CLAUDE.md`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "my-safety-checker --input-file $CLAUDE_TOOL_INPUT"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command", 
            "command": "prettier --write $CLAUDE_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

**23+ hook events** including:
- `PreToolUse` / `PostToolUse` — intercept any tool call
- `PreBashCommand` — inspect shell commands before execution
- `OnError` — custom error handling
- `OnSessionStart` / `OnSessionEnd`
- `OnContextCompaction` — triggered before compression

**Hook can return** `PermissionRequest` with 4 capabilities:
1. `APPROVE` — bypass normal permission check
2. `DENY` — block the operation
3. `MODIFY_INPUT` — mutate the tool's input parameters
4. `PROVIDE_REASON` — add explanation shown to user

### Multi-Agent Architecture (docs/07-multi-agent.md)

**3 coordination modes:**

```
Sub-Agent:
  Main Agent ──dispatch──► Sub Agent
                              │
                              └─ executes task
                              └─ returns result ──► Main Agent continues

Coordinator (pure orchestration):
  Coordinator ──task──► Agent A (reads files, writes code)
  Coordinator ──task──► Agent B (runs tests)
  Coordinator CANNOT read/write itself — enforces separation

Swarm (peer-to-peer):
  Agent "Alice" ◄──mailbox──► Agent "Bob"
  Agent "Bob"   ◄──mailbox──► Agent "Carol"
  Each agent independent, 3 execution backends
```

**Worktree isolation:** Each agent gets its own Git worktree copy to prevent concurrent file edit conflicts.

### Memory System (docs/08-memory-system.md)

**4 memory types:**
1. **In-context** — current conversation window
2. **External files** — `CLAUDE.md`, project docs injected at session start
3. **Semantic memory** — Sonnet-based recall with async prefetch
4. **Background extraction** — sub-agent runs after sessions to extract learnings

**Anti-drift:** Memory entries use a closed taxonomy (structured tags) to prevent semantic drift over many sessions.

### Code Editing Strategy (docs/05-code-editing-strategy.md)

Claude Code uses **search-and-replace over whole-file rewrite**:

```
Why search-and-replace:
  ✓ Smaller diffs → fewer tokens → lower cost
  ✓ Uniqueness constraint forces model to be precise
  ✓ Hallucination resistance (must match exact existing text)
  ✓ Preserves unchanged code exactly (no reformatting side effects)

Uniqueness constraint:
  The search string must appear EXACTLY ONCE in the file.
  If 0 matches → error, model must retry with more context.
  If 2+ matches → error, model must add surrounding lines.

Pre-edit read enforcement:
  Tool executor checks whether the file was read in current context.
  If not → forces a ReadFile call first, then allows edit.
```

---

## Building Your Own Agent (Minimal Components)

From `docs/12-minimal-components.md` — the 7 minimal components:

```typescript
// 1. Message loop
async function agentLoop(userMessage: string) {
  messages.push({ role: 'user', content: userMessage });
  
  while (true) {
    const response = await anthropic.messages.create({
      model: 'claude-opus-4-5',
      messages,
      tools: toolDefinitions,
      stream: true,
    });
    
    // 2. Stream handler
    for await (const chunk of response) {
      if (chunk.type === 'content_block_delta') {
        process.stdout.write(chunk.delta.text ?? '');
      }
    }
    
    // 3. Tool execution
    const toolUses = extractToolUses(response);
    if (toolUses.length === 0) break;  // done
    
    const results = await Promise.all(
      toolUses.map(t => executeTool(t.name, t.input))
    );
    
    // 4. Result injection
    messages.push({ role: 'assistant', content: response.content });
    messages.push({ role: 'user', content: results.map(toToolResult) });
  }
}

// 5. Tool registry
const tools = new Map<string, ToolHandler>();
tools.set('read_file', async ({ path }) => fs.readFile(path, 'utf8'));
tools.set('write_file', async ({ path, content }) => fs.writeFile(path, content));
tools.set('bash', async ({ command }) => execSync(command, { encoding: 'utf8' }));

// 6. Context management (basic)
function trimContext(messages: Message[], maxTokens: number) {
  // Estimate tokens, drop oldest non-system messages until within budget
  while (estimateTokens(messages) > maxTokens) {
    const firstNonSystem = messages.findIndex(m => m.role !== 'system');
    if (firstNonSystem === -1) break;
    messages.splice(firstNonSystem, 1);
  }
}

// 7. Permission check (minimal)
async function checkPermission(tool: string, input: unknown): Promise<boolean> {
  const dangerous = ['bash', 'write_file', 'delete_file'];
  if (!dangerous.includes(tool)) return true;
  const answer = await prompt(`Allow ${tool}? [y/N] `);
  return answer.toLowerCase() === 'y';
}
```

**Full 1,300-line implementation:** https://github.com/Windy3f3f3f3f/claude-code-from-scratch

---

## Performance Design Patterns

### 9-Phase Parallel Startup (~235ms critical path)

```
Phase 1-3 (parallel): Load config + Init MCP + Check git status
Phase 4-6 (parallel): Load CLAUDE.md + Prefetch memory + Warm tool registry  
Phase 7-8 (parallel): Init renderer + Authenticate
Phase 9 (sequential): Start agent loop
```

### Streaming Pipeline

```
API chunk arrives
      │
      ▼ (< 1ms)
Token renderer (Ink + Yoga Flexbox)
      │
      ▼
Terminal output
      │
      ├─ Meanwhile: parse partial JSON for tool calls
      └─ Tool pre-execution starts before stream ends
```

### Prompt Cache Strategy

Claude Code structures its system prompt for maximum cache hits:

```
[CACHED - rarely changes]
  System prompt core
  Tool definitions (all 66)
  CLAUDE.md content

[NOT CACHED - changes every turn]  
  Git status
  Current file contents
  Recent tool results
  User message
```

Cache break detection: If cache hit rate drops below threshold, system re-orders content blocks to restore caching.

---

## Common Patterns for AI Agent Development

### Reliable File Editing

```typescript
// Always read before edit — Claude Code enforces this at the executor level
async function safeEdit(path: string, search: string, replace: string) {
  const content = await fs.readFile(path, 'utf8');
  
  const matches = content.split(search).length - 1;
  if (matches === 0) throw new Error(`Search string not found in ${path}`);
  if (matches > 1) throw new Error(`Search string ambiguous (${matches} matches) — add more context`);
  
  return content.replace(search, replace);
}
```

### Context Window Management

```typescript
const COMPRESSION_THRESHOLDS = {
  TRUNCATE: 0.80,      // 80% full → truncate old tool outputs
  DEDUPLICATE: 0.85,   // 85% → deduplicate  
  FOLD: 0.90,          // 90% → fold inactive segments
  SUMMARIZE: 0.95,     // 95% → summarize entire history
};

async function manageContext(messages: Message[], tokenCount: number, maxTokens: number) {
  const ratio = tokenCount / maxTokens;
  
  if (ratio > COMPRESSION_THRESHOLDS.SUMMARIZE) {
    return await summarizeHistory(messages);  // sub-agent summarizes
  } else if (ratio > COMPRESSION_THRESHOLDS.FOLD) {
    return foldInactiveSegments(messages);
  } else if (ratio > COMPRESSION_THRESHOLDS.DEDUPLICATE) {
    return deduplicateMessages(messages);
  } else if (ratio > COMPRESSION_THRESHOLDS.TRUNCATE) {
    return truncateToolOutputs(messages);
  }
  return messages;
}
```

### Shell Safety (simplified from the 23-check AST analyzer)

```typescript
const DANGEROUS_PATTERNS = [
  /rm\s+-rf?\s+\/(?!\w)/,    // rm -rf /
  />\s*\/dev\/sd[a-z]/,      // write to block device
  /chmod\s+777/,              // world-writable
  /curl.*\|\s*bash/,          // curl pipe bash
  /eval\s*\(/,                // eval injection
];

function isSafeCommand(command: string): boolean {
  return !DANGEROUS_PATTERNS.some(p => p.test(command));
}
```

---

## Troubleshooting Common Agent Issues

| Problem | Claude Code's Solution | Your Implementation |
|---------|----------------------|---------------------|
| Context too long | 4-level progressive compression | Implement `trimContext()` with graceful degradation |
| Model stops mid-task | 7 Continue Sites, auto-retry | Check `stop_reason`, retry with `continue` message |
| Tool output too large | Auto-disk-offload at 100K chars | Write to temp file, pass path to model |
| Concurrent file edits | Git Worktree per agent | Use file locking or separate working directories |
| Hallucinated edits | Uniqueness constraint on search | Validate search string before applying edit |
| Slow startup | 9-phase parallel init | `Promise.all()` for independent initialization tasks |
| Dangerous commands | 5-layer defense + AST analysis | At minimum: pattern matching + user confirmation |

---

## Key Metrics from Source Analysis

| Metric | Value |
|--------|-------|
| Total source lines | 512,000+ |
| TypeScript files | 1,884 |
| Built-in tools | 66+ |
| Compression pipeline levels | 4 |
| Permission defense layers | 5 |
| Bash safety checks (AST) | 23 |
| Hook event types | 23+ |
| Startup critical path | ~235ms |
| Auto-restore files after compaction | 5 most recently edited |
| Confirmation debounce | 200ms |

---

## Related Resources

- **Source analysis docs:** https://windy3f3f3f3f.github.io/how-claude-code-works/#/
- **Build from scratch (1,300 lines, 8 chapters):** https://github.com/Windy3f3f3f3f/claude-code-from-scratch
- **Official Claude Code:** https://github.com/anthropics/claude-code
- **Anthropic API docs:** https://docs.anthropic.com
```
