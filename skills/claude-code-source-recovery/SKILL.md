```markdown
---
name: claude-code-source-recovery
description: Skill for navigating, understanding, and working with the recovered Claude Code 2.1.88 TypeScript source code extracted from the accidental npm source map upload.
triggers:
  - explore claude code source code
  - understand claude code architecture
  - navigate recovered claude code src
  - work with claude code internals
  - study claude code CLI structure
  - analyze claude code MCP implementation
  - read claude code command system
  - investigate claude code terminal UI
---

# Claude Code 2.1.88 Source Recovery

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

On 2026-03-31, Anthropic accidentally published `@anthropic-ai/claude-code@2.1.88` to npm with a `cli.js.map` source map file (~57MB) that contained the full TypeScript source in its `sourcesContent` field. After extraction, this yields ~700,000 lines of TypeScript code. This repository is the recovered and organized source tree for research and architectural study.

**This is NOT an official Anthropic release.** All copyright belongs to Anthropic.

---

## Repository Structure

```
src/
├── entrypoints/        # CLI bootstrap & initialization
├── commands/           # Command system (login, mcp, review, tasks, ...)
├── components/         # React + Ink terminal UI components
├── services/           # Core business logic (policy, sync, remote capabilities)
├── hooks/              # Interactive terminal state management
├── utils/              # Auth, file ops, process management helpers
└── ink/                # Custom terminal rendering infrastructure
```

---

## Installing the Leaked Version (from Tencent mirror cache)

The 2.1.88 version was pulled from npm. Use the Tencent CDN cache while it lasts:

```bash
npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz
```

---

## Source Recovery: How the Extraction Works

The source map format stores original sources in `sourcesContent`. To recover files:

```typescript
import * as fs from "fs";
import * as path from "path";

interface SourceMap {
  sources: string[];
  sourcesContent: (string | null)[];
}

function recoverSources(mapFilePath: string, outputDir: string): void {
  const raw = fs.readFileSync(mapFilePath, "utf-8");
  const map: SourceMap = JSON.parse(raw);

  map.sources.forEach((sourcePath, i) => {
    const content = map.sourcesContent[i];
    if (!content) return;

    // Normalize paths: strip leading ../ or webpack prefixes
    const normalized = sourcePath.replace(/^(\.\.\/)+/, "").replace(/^webpack:\/\/\//, "");
    const outPath = path.join(outputDir, normalized);

    fs.mkdirSync(path.dirname(outPath), { recursive: true });
    fs.writeFileSync(outPath, content, "utf-8");
    console.log(`Recovered: ${outPath}`);
  });
}

recoverSources("./cli.js.map", "./recovered-src");
```

Run with:

```bash
npx ts-node recover.ts
# or
node -e "require('./recover.js')"
```

---

## Key Architectural Patterns

### 1. Command System (`src/commands/`)

Commands are loaded dynamically and support built-ins, skills, plugins, and MCP commands:

```typescript
// Pattern observed in commands loader
interface Command {
  name: string;
  description: string;
  handler: (args: string[], context: CommandContext) => Promise<void>;
  aliases?: string[];
}

// Commands are registered via a central registry
const commandRegistry = new Map<string, Command>();

function registerCommand(cmd: Command): void {
  commandRegistry.set(cmd.name, cmd);
  cmd.aliases?.forEach((alias) => commandRegistry.set(alias, cmd));
}
```

Key commands discovered in source:
- `login` — OAuth and API key authentication
- `mcp` — Model Context Protocol server management
- `review` — Code review workflows
- `tasks` — Task/todo management
- `config` — Configuration management

### 2. Terminal UI with React + Ink (`src/components/`)

Claude Code renders its TUI using [Ink](https://github.com/vadimdemedes/ink) (React for CLIs):

```tsx
import React, { useState, useEffect } from "react";
import { Box, Text, useInput } from "ink";

// Typical component pattern from recovered source
const PromptInput: React.FC<{ onSubmit: (value: string) => void }> = ({ onSubmit }) => {
  const [input, setInput] = useState("");

  useInput((char, key) => {
    if (key.return) {
      onSubmit(input);
      setInput("");
    } else if (key.backspace || key.delete) {
      setInput((prev) => prev.slice(0, -1));
    } else if (char) {
      setInput((prev) => prev + char);
    }
  });

  return (
    <Box>
      <Text color="green">❯ </Text>
      <Text>{input}</Text>
    </Box>
  );
};
```

### 3. MCP (Model Context Protocol) Integration (`src/commands/mcp/`)

```typescript
// MCP server configuration pattern
interface MCPServerConfig {
  name: string;
  transport: "stdio" | "http" | "sse";
  command?: string;       // for stdio transport
  args?: string[];
  url?: string;           // for http/sse transport
  env?: Record<string, string>;
}

// MCP servers are stored in config and loaded at startup
async function loadMCPServers(configPath: string): Promise<MCPServerConfig[]> {
  const config = await readConfig(configPath);
  return config.mcpServers ?? [];
}
```

### 4. Authentication Utilities (`src/utils/`)

```typescript
// Auth pattern — uses env vars, never hardcoded keys
const API_KEY = process.env.ANTHROPIC_API_KEY;
const AUTH_TOKEN = process.env.CLAUDE_AUTH_TOKEN;

interface AuthState {
  apiKey?: string;
  oauthToken?: string;
  isAuthenticated: boolean;
}

async function getAuthState(): Promise<AuthState> {
  if (process.env.ANTHROPIC_API_KEY) {
    return { apiKey: process.env.ANTHROPIC_API_KEY, isAuthenticated: true };
  }
  // Fall back to stored OAuth token
  const stored = await readStoredToken();
  return stored
    ? { oauthToken: stored, isAuthenticated: true }
    : { isAuthenticated: false };
}
```

### 5. Feature Flags Pattern

Feature flags are baked at build time via `bun:bundle` macros:

```typescript
// Pattern seen throughout the source
import { define } from "bun:bundle"; // build-time constant

const FEATURE_TASKS_ENABLED = define("FEATURE_TASKS", false);
const FEATURE_REMOTE_MCP = define("FEATURE_REMOTE_MCP", true);

if (FEATURE_TASKS_ENABLED) {
  registerCommand(tasksCommand);
}
```

When studying the source, treat these as compile-time constants that may have been `true` or `false` at ship time.

---

## Navigating the Source

### Finding a Feature

```bash
# Search for a specific command implementation
grep -r "commandName.*review" src/commands/ --include="*.ts"

# Find all Ink components
find src/components -name "*.tsx" | head -20

# Locate MCP-related code
grep -r "ModelContextProtocol\|mcpServer\|MCPClient" src/ --include="*.ts" -l
```

### Understanding the Entry Point

```bash
# Start here to understand bootstrap
cat src/entrypoints/cli.ts
# or
cat src/entrypoints/index.ts
```

### Tracing a Command Flow

```
User types command
      ↓
src/entrypoints/cli.ts        # parses argv
      ↓
src/commands/index.ts         # routes to command handler
      ↓
src/commands/<name>/index.ts  # executes command logic
      ↓
src/components/<UI>.tsx       # renders output via Ink
      ↓
src/services/                 # business logic & API calls
```

---

## Attempting to Run the Recovered Source

> ⚠️ This requires significant reconstruction effort.

```bash
# 1. Initialize package.json
npm init -y

# 2. Install likely dependencies (inferred from imports in source)
npm install ink react @anthropic-ai/sdk commander chalk zod
npm install -D typescript @types/react @types/node bun

# 3. Compile TypeScript
npx tsc --strict --target ES2022 --module NodeNext

# 4. Handle bun:bundle macros by replacing with literals
# Find all `define(` calls and replace with hardcoded values

# 5. Run entry point
node dist/entrypoints/cli.js
```

---

## Common Patterns When Studying This Code

### Reading Service Layer

```typescript
// Services typically follow this shape
class SyncService {
  private config: Config;

  constructor(config: Config) {
    this.config = config;
  }

  async sync(): Promise<SyncResult> {
    // implementation
  }
}

// Instantiated in entrypoint or command handler:
const syncService = new SyncService(loadConfig());
await syncService.sync();
```

### Config File Location

```typescript
// Config is typically stored at:
const CONFIG_DIR = path.join(
  process.env.HOME ?? process.env.USERPROFILE ?? "~",
  ".claude"
);
const CONFIG_FILE = path.join(CONFIG_DIR, "config.json");
```

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| `bun:bundle` import errors | Bun-specific build macro | Replace `define(...)` calls with literal values |
| Missing type declarations | Incomplete extraction | Check `sourcesContent` for null entries (some files may be missing) |
| Path resolution failures | Source map paths may be relative | Normalize `../` prefixes during recovery |
| React/Ink version mismatch | Ink requires specific React version | Use `ink@4` with `react@18` |
| `ANTHROPIC_API_KEY` not found | Auth env var missing | `export ANTHROPIC_API_KEY=your_key_here` |

---

## Important Notes for AI Agents

1. **Do not reproduce or redistribute** substantial portions of this source — copyright belongs to Anthropic.
2. When helping a developer **understand** the architecture, reference file paths and patterns rather than copy-pasting large blocks.
3. The source uses **Bun** as its runtime/bundler — some APIs (`bun:bundle`, `Bun.file`, etc.) are Bun-specific.
4. The codebase is **TypeScript throughout** — all files are `.ts` or `.tsx`.
5. The **MCP implementation** here is a reference for how a production CLI integrates Model Context Protocol — useful for building similar tools.
6. Feature flag values at runtime (2.1.88) are unknown — analyze both branches when studying conditional code.

---

## References

- [npm package (archived)](https://www.npmjs.com/package/@anthropic-ai/claude-code/v/2.1.88)
- [Tencent mirror cache](https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz)
- [Ink — React for CLIs](https://github.com/vadimdemedes/ink)
- [Model Context Protocol spec](https://modelcontextprotocol.io)
- [Source Map spec (TC39)](https://tc39.es/source-map/)
```
