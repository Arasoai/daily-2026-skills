---
name: claude-code-source-analysis
description: Skill for exploring and understanding the recovered Claude Code 2.1.88 TypeScript source code structure, architecture, and implementation patterns.
triggers:
  - explore claude code source code
  - understand claude code architecture
  - analyze claude code implementation
  - study claude code cli structure
  - look at claude code mcp implementation
  - examine claude code components
  - understand how claude code works internally
  - navigate claude code recovered source
---

# Claude Code 2.1.88 Source Recovery

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

This project contains the recovered TypeScript source code of `@anthropic-ai/claude-code` version 2.1.88, extracted from the accidentally published `cli.js.map` source map file (57MB). The restored codebase is ~700,000 lines of TypeScript organized into a clean directory structure.

## How the Source Was Recovered

The `cli.js.map` file contained a `sourcesContent` field with all original source files. The recovery process:

```typescript
// Conceptual recovery process
import * as fs from 'fs';

const sourceMap = JSON.parse(fs.readFileSync('cli.js.map', 'utf-8'));

// sourceMap.sources = array of original file paths
// sourceMap.sourcesContent = array of original file contents

sourceMap.sources.forEach((filePath: string, index: number) => {
  const content = sourceMap.sourcesContent[index];
  if (content) {
    fs.mkdirSync(path.dirname(filePath), { recursive: true });
    fs.writeFileSync(filePath, content);
  }
});
```

## Installing the Original Package (Mirror)

Since v2.1.88 was pulled from npm, use the Tencent mirror cache:

```bash
npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz
```

## Project Structure

```
src/
├── entrypoints/        # CLI entry points and initialization
│   ├── cli.ts          # Main CLI bootstrap
│   └── index.ts        # Module exports
├── commands/           # Command system
│   ├── login.ts        # Authentication command
│   ├── mcp.ts          # MCP protocol command
│   ├── review.ts       # Code review command
│   └── tasks.ts        # Task management command
├── components/         # React + Ink terminal UI components
│   ├── Chat.tsx        # Main chat interface
│   ├── Message.tsx     # Message rendering
│   └── Prompt.tsx      # Input prompt component
├── services/           # Core business logic
│   ├── policy.ts       # Policy/permission management
│   ├── sync.ts         # State synchronization
│   └── remote.ts       # Remote capability handling
├── hooks/              # React hooks for terminal state
│   ├── useInput.ts     # Input handling hook
│   └── useSession.ts   # Session state hook
├── utils/              # Utility functions
│   ├── auth.ts         # Authentication utilities
│   ├── file.ts         # File operations
│   └── process.ts      # Process management
└── ink/                # Custom terminal rendering infrastructure
```

## Key Architecture Patterns

### Command System

Claude Code uses a hybrid command loading mechanism:

```typescript
// Pattern: Command registration
interface Command {
  name: string;
  description: string;
  aliases?: string[];
  handler: (args: string[], context: CommandContext) => Promise<void>;
}

// Built-in commands are registered at startup
const builtinCommands: Command[] = [
  {
    name: 'login',
    description: 'Authenticate with Anthropic',
    handler: async (args, ctx) => {
      // OAuth flow or API key setup
      const apiKey = process.env.ANTHROPIC_API_KEY;
      await ctx.auth.initialize(apiKey);
    }
  },
  {
    name: 'mcp',
    description: 'Manage MCP servers',
    aliases: ['mcp-server'],
    handler: async (args, ctx) => {
      await ctx.mcp.handleCommand(args);
    }
  }
];
```

### Terminal UI with React + Ink

```typescript
import React, { useState, useEffect } from 'react';
import { Box, Text, useInput, useApp } from 'ink';

// Pattern: Ink component for terminal rendering
const ChatInterface: React.FC<{ session: Session }> = ({ session }) => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [inputValue, setInputValue] = useState('');
  const { exit } = useApp();

  useInput((input, key) => {
    if (key.escape) {
      exit();
    }
    if (key.return) {
      handleSubmit(inputValue);
      setInputValue('');
    } else {
      setInputValue(prev => prev + input);
    }
  });

  return (
    <Box flexDirection="column">
      {messages.map((msg, i) => (
        <Box key={i} marginBottom={1}>
          <Text color={msg.role === 'assistant' ? 'cyan' : 'white'}>
            {msg.content}
          </Text>
        </Box>
      ))}
      <Box>
        <Text color="green">{'> '}</Text>
        <Text>{inputValue}</Text>
      </Box>
    </Box>
  );
};
```

### MCP (Model Context Protocol) Integration

```typescript
// Pattern: MCP server connection
interface MCPServerConfig {
  name: string;
  command: string;
  args?: string[];
  env?: Record<string, string>;
}

class MCPClient {
  private servers: Map<string, MCPServer> = new Map();

  async connectServer(config: MCPServerConfig): Promise<void> {
    const server = new MCPServer({
      command: config.command,
      args: config.args ?? [],
      env: {
        ...process.env,
        ...config.env
      }
    });

    await server.initialize();
    this.servers.set(config.name, server);
  }

  async listTools(serverName: string): Promise<Tool[]> {
    const server = this.servers.get(serverName);
    if (!server) throw new Error(`Server ${serverName} not connected`);
    return server.listTools();
  }

  async callTool(serverName: string, toolName: string, params: unknown): Promise<ToolResult> {
    const server = this.servers.get(serverName);
    if (!server) throw new Error(`Server ${serverName} not connected`);
    return server.callTool(toolName, params);
  }
}
```

### Authentication Pattern

```typescript
// Pattern: API key management
class AuthService {
  private apiKey: string | null = null;

  async initialize(key?: string): Promise<void> {
    // Priority: explicit key > env var > stored config
    this.apiKey = key
      ?? process.env.ANTHROPIC_API_KEY
      ?? await this.loadStoredKey();

    if (!this.apiKey) {
      throw new Error('No API key found. Set ANTHROPIC_API_KEY or run: claude login');
    }
  }

  private async loadStoredKey(): Promise<string | null> {
    const configPath = path.join(os.homedir(), '.claude', 'config.json');
    try {
      const config = JSON.parse(fs.readFileSync(configPath, 'utf-8'));
      return config.apiKey ?? null;
    } catch {
      return null;
    }
  }

  getHeaders(): Record<string, string> {
    return {
      'x-api-key': this.apiKey!,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json'
    };
  }
}
```

### Feature Flags Pattern

```typescript
// Pattern: Build-time feature flags (seen throughout source)
const FEATURES = {
  REMOTE_CAPABILITY: process.env.CLAUDE_FEATURE_REMOTE === 'true',
  MCP_ENABLED: process.env.CLAUDE_MCP_ENABLED !== 'false',
  TASKS_ENABLED: process.env.CLAUDE_TASKS_ENABLED === 'true',
  DEBUG_MODE: process.env.CLAUDE_DEBUG === 'true',
} as const;

function withFeatureFlag<T>(
  flag: keyof typeof FEATURES,
  handler: () => T,
  fallback: T
): T {
  return FEATURES[flag] ? handler() : fallback;
}
```

### Session and State Management

```typescript
// Pattern: Session hook
function useSession() {
  const [session, setSession] = useState<Session | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const startSession = async (config: SessionConfig) => {
    setIsLoading(true);
    try {
      const newSession = await Session.create({
        apiKey: process.env.ANTHROPIC_API_KEY!,
        model: config.model ?? 'claude-opus-4-5',
        systemPrompt: config.systemPrompt,
        tools: config.tools ?? []
      });
      setSession(newSession);
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
    } finally {
      setIsLoading(false);
    }
  };

  return { session, isLoading, error, startSession };
}
```

## Exploring the Source

```bash
# Clone the recovery repository
git clone https://github.com/ponponon/claude_code_src.git
cd claude_code_src

# Count total lines
find src/ -name "*.ts" -o -name "*.tsx" | xargs wc -l | tail -1

# Find all command handlers
grep -r "handler:" src/commands/ --include="*.ts" -l

# Explore MCP implementation
find src/ -name "*mcp*" -type f

# Search for Ink component usage
grep -r "from 'ink'" src/ --include="*.tsx" -l

# Find all feature flag checks
grep -r "FEATURES\." src/ --include="*.ts" -l
```

## Key Dependencies (Inferred from Source)

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.x.x",
    "ink": "^4.x.x",
    "react": "^18.x.x",
    "@modelcontextprotocol/sdk": "^1.x.x",
    "commander": "^11.x.x",
    "chalk": "^5.x.x",
    "zod": "^3.x.x"
  },
  "devDependencies": {
    "typescript": "^5.x.x",
    "bun": "^1.x.x",
    "@types/react": "^18.x.x"
  }
}
```

## Reconstructing to Run

```bash
# 1. Initialize package
npm init -y

# 2. Install inferred dependencies
npm install @anthropic-ai/sdk ink react @modelcontextprotocol/sdk commander chalk zod
npm install -D typescript @types/react @types/node bun-types

# 3. Create tsconfig.json
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"]
}
EOF

# 4. Set environment variables
export ANTHROPIC_API_KEY="your-key-from-env"

# 5. Attempt build (note: bun:bundle macros may need replacement)
bun build src/entrypoints/cli.ts --outfile dist/cli.js
```

## Common Research Tasks

```bash
# Find how tools are registered
grep -r "registerTool\|addTool\|tools\.push" src/ --include="*.ts"

# Understand streaming implementation
grep -r "stream\|EventSource\|ReadableStream" src/ --include="*.ts" -l

# Find retry/error handling logic
grep -r "retry\|exponentialBackoff\|withRetry" src/ --include="*.ts"

# Locate permission/policy checks
grep -r "hasPermission\|checkPolicy\|canExecute" src/ --include="*.ts"

# Find all slash commands
grep -r "\/[a-z]" src/commands/ --include="*.ts"
```

## Troubleshooting

**`bun:bundle` macro errors**: These are Bun-specific build macros. Replace with runtime equivalents or stub them:
```typescript
// Replace: import { version } from 'bun:bundle';
const version = process.env.npm_package_version ?? '2.1.88';
```

**Missing type declarations**: Some internal types may reference Anthropic-internal packages. Create stub declaration files:
```typescript
// types/stubs.d.ts
declare module '@anthropic-ai/internal-utils' {
  export function someUtil(): void;
}
```

**React/Ink version conflicts**: Pin to exact versions used circa early 2026:
```bash
npm install ink@4.4.1 react@18.3.1
```

## Legal Notice

This repository contains source code recovered via source map. All intellectual property rights belong to Anthropic. This is for **research and educational purposes only**. Do not use for commercial purposes without proper licensing.
