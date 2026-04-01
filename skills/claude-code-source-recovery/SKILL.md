```markdown
---
name: claude-code-source-recovery
description: Skill for exploring and understanding the recovered Claude Code 2.1.88 TypeScript source code, its CLI architecture, command system, and MCP implementation.
triggers:
  - explore claude code source code
  - understand claude code architecture
  - how does claude code cli work
  - claude code command system internals
  - claude code mcp implementation
  - analyze claude code source map recovery
  - claude code ink react terminal ui
  - claude code 2.1.88 source structure
---

# Claude Code 2.1.88 Source Recovery

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

This repository contains the recovered TypeScript source code of `@anthropic-ai/claude-code` version 2.1.88, extracted from the accidentally published `cli.js.map` source map file (57MB). The recovered source spans ~700,000 lines of TypeScript and reveals the full CLI architecture, React/Ink terminal UI system, MCP integration, and command infrastructure.

**Key facts:**
- Source was extracted from the `sourcesContent` field of `cli.js.map`
- Original npm package `@anthropic-ai/claude-code@2.1.88` has been unpublished
- Tencent mirror cache still hosts the original tarball
- This is for **research and educational purposes only** — copyright belongs to Anthropic

---

## Installing the Original 2.1.88 Package (via Mirror)

Since the official npm version was pulled, use the Tencent mirror cache:

```bash
npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz
```

---

## Repository Structure

```
src/
├── entrypoints/        # CLI entry points and initialization
├── commands/           # Command system (login, mcp, review, tasks, etc.)
├── components/         # React + Ink terminal UI components
├── services/           # Core business logic (strategy, sync, remote capabilities)
├── hooks/              # Interactive terminal state management
├── utils/              # Auth, file ops, process management utilities
└── ink/                # Custom terminal rendering infrastructure
```

---

## Key Architectural Patterns

### 1. CLI Entry Point Pattern

```typescript
// src/entrypoints/cli.ts (typical pattern)
import { render } from 'ink';
import React from 'react';
import { AppRoot } from '../components/AppRoot';

async function main() {
  const { waitUntilExit } = render(
    React.createElement(AppRoot, { argv: process.argv.slice(2) })
  );
  await waitUntilExit();
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

### 2. Command Registration Pattern

Commands are registered with a descriptor object and a handler:

```typescript
// src/commands/login.ts (illustrative pattern)
import type { Command } from '../types/command';

export const loginCommand: Command = {
  name: 'login',
  description: 'Authenticate with Anthropic',
  aliases: [],
  async run(args, context) {
    const { config } = context;
    // OAuth or API key flow
    const apiKey = await promptForApiKey();
    await config.set('apiKey', apiKey);
    console.log('Logged in successfully.');
  },
};
```

### 3. React + Ink Terminal UI

Claude Code uses [Ink](https://github.com/vadimdemedes/ink) for rendering React components in the terminal:

```typescript
// src/components/ChatView.tsx (illustrative)
import React, { useState } from 'react';
import { Box, Text, useInput } from 'ink';

export function ChatView({ messages }: { messages: Message[] }) {
  return (
    <Box flexDirection="column">
      {messages.map((msg, i) => (
        <Box key={i} marginBottom={1}>
          <Text color={msg.role === 'assistant' ? 'cyan' : 'white'}>
            {msg.role === 'assistant' ? '🤖 Claude' : '👤 You'}: {msg.content}
          </Text>
        </Box>
      ))}
    </Box>
  );
}
```

### 4. MCP (Model Context Protocol) Integration

```typescript
// src/services/mcp.ts (illustrative pattern)
import { MCPClient } from '../mcp/client';

export class MCPService {
  private clients: Map<string, MCPClient> = new Map();

  async connectServer(name: string, config: MCPServerConfig) {
    const client = new MCPClient(config);
    await client.connect();
    this.clients.set(name, client);
    return client;
  }

  async listTools(serverName: string) {
    const client = this.clients.get(serverName);
    if (!client) throw new Error(`MCP server '${serverName}' not connected`);
    return client.listTools();
  }

  async callTool(serverName: string, toolName: string, args: unknown) {
    const client = this.clients.get(serverName);
    if (!client) throw new Error(`MCP server '${serverName}' not connected`);
    return client.callTool(toolName, args);
  }
}
```

### 5. Command Loader (Mixed Sources)

Claude Code supports built-in commands, dynamic skills, plugins, and MCP commands:

```typescript
// src/commands/loader.ts (illustrative)
export async function loadAllCommands(context: AppContext): Promise<Command[]> {
  const builtins = await loadBuiltinCommands();
  const skills   = await loadSkillCommands(context.skillsDir);
  const plugins  = await loadPluginCommands(context.pluginsDir);
  const mcp      = await loadMCPCommands(context.mcpService);

  return [...builtins, ...skills, ...plugins, ...mcp];
}
```

### 6. Feature Flags Pattern

Feature flags are baked in at build time via `bun:bundle` macros:

```typescript
// src/utils/features.ts (illustrative)
// These are replaced at build time — do not expect runtime toggling
export const FEATURE_REMOTE_TASKS  = process.env.FEATURE_REMOTE_TASKS  === 'true';
export const FEATURE_MCP_V2        = process.env.FEATURE_MCP_V2        === 'true';
export const FEATURE_REVIEW_CMD    = process.env.FEATURE_REVIEW_CMD    === 'true';
```

### 7. Auth / API Key Utilities

```typescript
// src/utils/auth.ts (illustrative)
export function getApiKey(): string {
  const key = process.env.ANTHROPIC_API_KEY
    ?? readStoredApiKey();  // reads from ~/.claude/config.json
  if (!key) {
    throw new Error(
      'No API key found. Run `claude login` or set ANTHROPIC_API_KEY.'
    );
  }
  return key;
}
```

---

## Recovering Source From cli.js.map Yourself

```typescript
import fs from 'fs';
import path from 'path';

// 1. Load the source map
const raw = fs.readFileSync('cli.js.map', 'utf8');
const map = JSON.parse(raw);

// 2. Iterate sources + sourcesContent
map.sources.forEach((sourcePath: string, i: number) => {
  const content = map.sourcesContent?.[i];
  if (!content) return;

  // Normalize path (strip webpack:// or similar prefixes)
  const normalized = sourcePath.replace(/^.*?\/src\//, 'recovered/src/');
  const outPath = path.resolve(normalized);

  fs.mkdirSync(path.dirname(outPath), { recursive: true });
  fs.writeFileSync(outPath, content, 'utf8');
  console.log(`Wrote: ${outPath}`);
});

console.log('Recovery complete.');
```

Run with:
```bash
npx tsx recover.ts
# or
bun recover.ts
```

---

## Exploring the Source: Common Entry Points

| Area | Path | What to Look For |
|---|---|---|
| CLI bootstrap | `src/entrypoints/` | `main()`, arg parsing, Ink render |
| Commands | `src/commands/` | `login`, `mcp`, `review`, `tasks`, `config` |
| Terminal UI | `src/components/` | `AppRoot`, `ChatView`, `Spinner`, `StatusBar` |
| MCP client | `src/services/mcp.ts` | Tool discovery, server lifecycle |
| Auth flow | `src/utils/auth.ts` | API key storage, OAuth tokens |
| Hooks | `src/hooks/` | `useChat`, `useInput`, `useConfig` |
| Ink layer | `src/ink/` | Custom renderers, layout primitives |

---

## Attempting to Run the Recovered Source

> ⚠️ The recovered source depends on internal Anthropic build tooling (`bun:bundle` macros, feature flags). Full execution is non-trivial.

```bash
# 1. Install dependencies (you must infer them from imports)
npm init -y
npm install ink react @anthropic-ai/sdk commander

# 2. Add tsconfig
npx tsc --init

# 3. Strip or stub bun:bundle macros
# Find all: grep -r "bun:bundle" src/

# 4. Try running an entrypoint
npx tsx src/entrypoints/cli.ts --help
```

---

## Environment Variables Referenced in Source

```bash
ANTHROPIC_API_KEY=sk-ant-...         # Primary API key
ANTHROPIC_BASE_URL=https://...       # Override API base URL
CLAUDE_CONFIG_DIR=~/.claude          # Config directory override
FEATURE_MCP_V2=true                  # Enable MCP v2 (build-time flag)
FEATURE_REMOTE_TASKS=true            # Enable remote tasks (build-time flag)
DEBUG=claude:*                       # Enable debug logging
```

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `Cannot find module 'bun:bundle'` | Bun-specific macro | Stub it: `export const bundle = {}` |
| `ink` render issues | Wrong React version | Pin `react@18`, `ink@4` or `ink@5` |
| Missing source in recovery | `sourcesContent` entry is `null` | Those files were already minified/excluded |
| `ANTHROPIC_API_KEY` not found | Env not set | `export ANTHROPIC_API_KEY=$YOUR_KEY` |
| TypeScript path errors | Inferred `tsconfig` paths wrong | Add `paths` aliases matching `src/` layout |

---

## Legal Notice

This skill documents a research project. The recovered source code is the intellectual property of **Anthropic**. Use for personal study and architecture research only. Do not redistribute, commercialize, or deploy the recovered code. See the repository's disclaimer for full details.
```
