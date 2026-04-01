```markdown
---
name: claude-code-source-recovery
description: Skill for navigating, understanding, and working with the recovered Claude Code 2.1.88 TypeScript source code extracted from the npm source map
triggers:
  - explore claude code source code
  - understand claude code architecture
  - work with recovered claude code src
  - navigate claude code commands
  - study claude code mcp implementation
  - analyze claude code terminal UI
  - read claude code hooks and services
  - build on top of claude code source
---

# Claude Code 2.1.88 Source Recovery

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

This repository contains the recovered TypeScript source code of `@anthropic-ai/claude-code` version 2.1.88. On 2026-03-31, Anthropic accidentally published the npm package with a `cli.js.map` source map file (~57MB) containing the full source in its `sourcesContent` field. After extraction and reconstruction, the result is ~700,000 lines of TypeScript across a structured `src/` directory.

The source reveals Claude Code's internal architecture: a React/Ink-based terminal UI, a rich command system, MCP (Model Context Protocol) integration, hooks-based state management, and a services layer.

---

## Installing the Leaked Package (from Tencent Mirror Cache)

The 2.1.88 version was pulled from official npm. Use the Tencent CDN cache:

```bash
npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz
```

Or clone and explore the recovered source:

```bash
git clone https://github.com/ponponon/claude_code_src.git
cd claude_code_src
```

---

## Repository Structure

```
src/
├── entrypoints/       # CLI entry points and bootstrapping
├── commands/          # Command definitions (login, mcp, review, tasks, …)
├── components/        # React + Ink terminal UI components
├── services/          # Core business logic (policy, sync, remote capabilities)
├── hooks/             # Terminal interaction state management
├── utils/             # Auth, file ops, process management
└── ink/               # Custom terminal rendering infrastructure
```

---

## Key Architectural Patterns

### 1. Entry Point Bootstrap (`src/entrypoints/`)

The CLI initializes by loading commands, resolving config, and mounting the Ink React tree.

```typescript
// src/entrypoints/cli.ts (reconstructed pattern)
import { render } from 'ink';
import React from 'react';
import { App } from '../components/App';
import { loadCommands } from '../commands';
import { resolveConfig } from '../utils/config';

async function main() {
  const config = await resolveConfig();
  const commands = await loadCommands(config);

  render(React.createElement(App, { commands, config }));
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

### 2. Command System (`src/commands/`)

Commands are loaded from multiple sources: built-ins, dynamic skills, plugins, and MCP commands.

```typescript
// Pattern for defining a built-in command
export interface Command {
  name: string;
  description: string;
  aliases?: string[];
  hidden?: boolean;
  handler: (args: string[], ctx: CommandContext) => Promise<void>;
}

// Example: a simple built-in command
export const loginCommand: Command = {
  name: 'login',
  description: 'Authenticate with Anthropic',
  async handler(args, ctx) {
    const token = process.env.ANTHROPIC_API_KEY;
    if (!token) {
      ctx.output.error('Set ANTHROPIC_API_KEY environment variable');
      process.exit(1);
    }
    await ctx.services.auth.login(token);
    ctx.output.success('Logged in successfully');
  },
};
```

### 3. Terminal UI with React + Ink (`src/components/`)

Claude Code renders its entire TUI using [Ink](https://github.com/vadimdemedes/ink), treating terminal output as a React component tree.

```typescript
// src/components/StatusBar.tsx (reconstructed pattern)
import React from 'react';
import { Box, Text } from 'ink';

interface StatusBarProps {
  model: string;
  tokenCount: number;
  isStreaming: boolean;
}

export const StatusBar: React.FC<StatusBarProps> = ({
  model,
  tokenCount,
  isStreaming,
}) => (
  <Box borderStyle="single" paddingX={1}>
    <Text color="cyan">{model}</Text>
    <Text> | Tokens: {tokenCount}</Text>
    {isStreaming && <Text color="yellow"> ⟳ streaming…</Text>}
  </Box>
);
```

### 4. Hooks for State Management (`src/hooks/`)

Interactive terminal state is managed via React hooks, similar to a web app.

```typescript
// src/hooks/useConversation.ts (reconstructed pattern)
import { useState, useCallback } from 'react';

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

export function useConversation() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  const sendMessage = useCallback(async (content: string) => {
    setIsLoading(true);
    setMessages((prev) => [...prev, { role: 'user', content }]);

    try {
      const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'x-api-key': process.env.ANTHROPIC_API_KEY!,
          'anthropic-version': '2023-06-01',
          'content-type': 'application/json',
        },
        body: JSON.stringify({
          model: 'claude-opus-4-5',
          max_tokens: 8096,
          messages,
        }),
      });
      const data = await response.json();
      setMessages((prev) => [
        ...prev,
        { role: 'assistant', content: data.content[0].text },
      ]);
    } finally {
      setIsLoading(false);
    }
  }, [messages]);

  return { messages, isLoading, sendMessage };
}
```

### 5. MCP Integration (`src/commands/mcp.ts`)

Model Context Protocol commands manage MCP server connections.

```typescript
// Reconstructed MCP command pattern
import { Command } from '../types';
import { MCPClient } from '../services/mcp';

export const mcpCommand: Command = {
  name: 'mcp',
  description: 'Manage MCP servers',
  async handler(args, ctx) {
    const [subcommand, ...rest] = args;

    switch (subcommand) {
      case 'add':
        await ctx.services.mcp.addServer({
          name: rest[0],
          transport: 'stdio',
          command: rest[1],
          args: rest.slice(2),
        });
        break;

      case 'list':
        const servers = await ctx.services.mcp.listServers();
        servers.forEach((s) => ctx.output.info(`• ${s.name} (${s.status})`));
        break;

      case 'remove':
        await ctx.services.mcp.removeServer(rest[0]);
        break;

      default:
        ctx.output.error(`Unknown mcp subcommand: ${subcommand}`);
    }
  },
};
```

### 6. Services Layer (`src/services/`)

```typescript
// src/services/auth.ts (reconstructed pattern)
export class AuthService {
  private tokenKey = 'ANTHROPIC_API_KEY';

  async login(token: string): Promise<void> {
    // Persists token to ~/.claude/config.json
    await this.persistToken(token);
  }

  getToken(): string | undefined {
    return process.env[this.tokenKey];
  }

  isAuthenticated(): boolean {
    return Boolean(this.getToken());
  }

  private async persistToken(token: string): Promise<void> {
    const fs = await import('fs/promises');
    const os = await import('os');
    const path = await import('path');
    const configDir = path.join(os.homedir(), '.claude');
    await fs.mkdir(configDir, { recursive: true });
    await fs.writeFile(
      path.join(configDir, 'config.json'),
      JSON.stringify({ apiKey: token }, null, 2),
    );
  }
}
```

---

## Extracting Source from cli.js.map Yourself

If you have the original package, extract sources programmatically:

```typescript
import fs from 'fs';
import path from 'path';

const mapFile = fs.readFileSync('cli.js.map', 'utf-8');
const sourceMap = JSON.parse(mapFile);

const { sources, sourcesContent } = sourceMap;

for (let i = 0; i < sources.length; i++) {
  const sourcePath = sources[i].replace(/^webpack:\/\/\//, '');
  const content = sourcesContent[i];
  if (!content) continue;

  const outPath = path.join('recovered', sourcePath);
  fs.mkdirSync(path.dirname(outPath), { recursive: true });
  fs.writeFileSync(outPath, content, 'utf-8');
}

console.log(`Recovered ${sources.length} source files`);
```

---

## Feature Flags Pattern

The source uses compile-time feature flags (`bun:bundle` macros):

```typescript
// Pattern seen throughout the source
declare const __FEATURE_MCP_ENABLED__: boolean;
declare const __FEATURE_REMOTE_TASKS__: boolean;

if (__FEATURE_MCP_ENABLED__) {
  commands.push(mcpCommand);
}

if (__FEATURE_REMOTE_TASKS__) {
  commands.push(tasksCommand);
}
```

When attempting to build, stub these globals:

```typescript
// build-stubs.ts
(globalThis as any).__FEATURE_MCP_ENABLED__ = true;
(globalThis as any).__FEATURE_REMOTE_TASKS__ = true;
```

---

## Environment Variables

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Required for all API calls |
| `CLAUDE_MODEL` | Override default model |
| `CLAUDE_CONFIG_DIR` | Override `~/.claude` config directory |
| `CLAUDE_DEBUG` | Enable verbose debug logging |
| `MCP_SERVER_URL` | Override MCP server endpoint |

---

## Attempting to Run the Recovered Source

```bash
# 1. Initialize package
cd claude_code_src
npm init -y

# 2. Install core dependencies (inferred from source imports)
npm install ink react @anthropic-ai/sdk commander zod

npm install -D typescript @types/react @types/node tsx

# 3. Create tsconfig
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "jsx": "react",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
EOF

# 4. Stub feature flags and attempt compilation
npx tsx src/entrypoints/cli.ts
```

---

## Navigating the Codebase

```bash
# Find all command definitions
grep -r "export const.*Command" src/commands/ --include="*.ts" -l

# Find MCP-related code
grep -r "MCP\|ModelContext\|mcp" src/ --include="*.ts" -l

# Find all Ink components
grep -r "from 'ink'" src/ --include="*.tsx" -l

# Find auth/token handling
grep -r "ANTHROPIC_API_KEY\|apiKey\|token" src/utils/ --include="*.ts" -l

# Find feature flags
grep -r "__FEATURE_" src/ --include="*.ts"
```

---

## Common Patterns in the Source

### Streaming responses

```typescript
// Pattern for streaming from Anthropic API
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

async function* streamResponse(userMessage: string) {
  const stream = await client.messages.stream({
    model: 'claude-opus-4-5',
    max_tokens: 8096,
    messages: [{ role: 'user', content: userMessage }],
  });

  for await (const chunk of stream) {
    if (
      chunk.type === 'content_block_delta' &&
      chunk.delta.type === 'text_delta'
    ) {
      yield chunk.delta.text;
    }
  }
}
```

### Config file access

```typescript
import fs from 'fs/promises';
import os from 'os';
import path from 'path';

const CONFIG_DIR = process.env.CLAUDE_CONFIG_DIR ?? path.join(os.homedir(), '.claude');
const CONFIG_FILE = path.join(CONFIG_DIR, 'config.json');

async function readConfig(): Promise<Record<string, unknown>> {
  try {
    const raw = await fs.readFile(CONFIG_FILE, 'utf-8');
    return JSON.parse(raw);
  } catch {
    return {};
  }
}
```

---

## Troubleshooting

**`bun:bundle` macro errors during build**
- These are Bun-specific bundler macros. Stub them as constants or use Bun to build:
  ```bash
  bun build src/entrypoints/cli.ts --outdir dist
  ```

**Missing module errors**
- The source has many inferred dependencies. Check the import statement and install the corresponding npm package.

**Feature flag `ReferenceError`**
- Add the stubs from the "Feature Flags" section above before running.

**Ink rendering issues in non-TTY environments**
- Set `CI=true` to disable interactive rendering, or redirect stderr:
  ```bash
  FORCE_COLOR=0 npx tsx src/entrypoints/cli.ts 2>/dev/null
  ```

---

## Legal Notice

This project is for archival and educational research only. All original code is copyright Anthropic. Do not use for commercial purposes without legal review.
```
