```markdown
---
name: cc-haha-claude-code-local
description: Run a locally patched version of the leaked Claude Code source with any Anthropic-compatible API
triggers:
  - set up claude code locally
  - run claude code with custom api
  - use minimax with claude code
  - configure anthropic base url for claude code
  - fix claude code leaked source
  - connect openrouter to claude code
  - run claude code tui locally
  - use third party model with claude code
---

# cc-haha: Claude Code Local Runner

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`cc-haha` is a patched, locally runnable version of the Claude Code source code leaked from Anthropic's npm registry (2026-03-31). It fixes multiple startup blockers in the original leaked source and adds support for any Anthropic-compatible API endpoint (MiniMax, OpenRouter, LiteLLM proxies, etc.), exposing the full Ink TUI interactive interface.

---

## Installation

### Prerequisites

**Bun** (required runtime):

```bash
# macOS / Linux
curl -fsSL https://bun.sh/install | bash

# macOS via Homebrew
brew install bun

# Windows (PowerShell)
powershell -c "irm bun.sh/install.ps1 | iex"

# Verify
bun --version
```

On minimal Linux, install `unzip` first if prompted:

```bash
apt update && apt install -y unzip
```

**Windows only**: Install [Git for Windows](https://git-scm.com/download/win) — the project's internal shell execution depends on Git Bash.

### Clone and Install

```bash
git clone https://github.com/NanmiCoder/cc-haha.git
cd cc-haha
bun install
```

---

## Configuration

Copy the example env file and fill in your credentials:

```bash
cp .env.example .env
```

### `.env` Reference

```env
# Authentication — pick one:
ANTHROPIC_API_KEY=your_key_here          # sent as x-api-key header
ANTHROPIC_AUTH_TOKEN=your_token_here     # sent as Authorization: Bearer header

# Custom API endpoint (optional — defaults to official Anthropic)
ANTHROPIC_BASE_URL=https://api.minimaxi.com/anthropic

# Model routing
ANTHROPIC_MODEL=MiniMax-M2.7-highspeed
ANTHROPIC_DEFAULT_SONNET_MODEL=MiniMax-M2.7-highspeed
ANTHROPIC_DEFAULT_HAIKU_MODEL=MiniMax-M2.7-highspeed
ANTHROPIC_DEFAULT_OPUS_MODEL=MiniMax-M2.7-highspeed

# Timeouts and telemetry
API_TIMEOUT_MS=3000000
DISABLE_TELEMETRY=1
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

### Alternative: `~/.claude/settings.json`

You can also configure via the standard Claude Code settings file:

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "$ANTHROPIC_AUTH_TOKEN",
    "ANTHROPIC_BASE_URL": "https://api.minimaxi.com/anthropic",
    "ANTHROPIC_MODEL": "MiniMax-M2.7-highspeed",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "MiniMax-M2.7-highspeed",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "MiniMax-M2.7-highspeed",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "MiniMax-M2.7-highspeed"
  }
}
```

**Priority**: Shell environment variables > `.env` file > `~/.claude/settings.json`

---

## Key Commands

### macOS / Linux

```bash
# Full interactive TUI
./bin/claude-haha

# Headless / print mode (single prompt, great for scripts/CI)
./bin/claude-haha -p "explain the architecture of this codebase"

# Pipe input
echo "review this function for bugs" | ./bin/claude-haha -p

# Show all CLI options
./bin/claude-haha --help

# Force recovery/fallback CLI (if TUI fails)
CLAUDE_CODE_FORCE_RECOVERY_CLI=1 ./bin/claude-haha
```

### Windows (PowerShell / cmd)

```powershell
# Interactive TUI
bun --env-file=.env ./src/entrypoints/cli.tsx

# Headless mode
bun --env-file=.env ./src/entrypoints/cli.tsx -p "your prompt here"

# Recovery CLI fallback
bun --env-file=.env ./src/localRecoveryCli.ts
```

### Windows (Git Bash)

```bash
# Identical to macOS/Linux
./bin/claude-haha
./bin/claude-haha -p "your prompt here"
```

---

## API Provider Configuration Examples

### MiniMax

```env
ANTHROPIC_BASE_URL=https://api.minimaxi.com/anthropic
ANTHROPIC_AUTH_TOKEN=$MINIMAX_API_KEY
ANTHROPIC_MODEL=MiniMax-M2.7-highspeed
```

### OpenRouter

```env
ANTHROPIC_BASE_URL=https://openrouter.ai/api
ANTHROPIC_AUTH_TOKEN=$OPENROUTER_API_KEY
ANTHROPIC_MODEL=anthropic/claude-3.5-sonnet
```

> ⚠️ OpenRouter: use `https://openrouter.ai/api` NOT `https://openrouter.ai/anthropic` — the latter returns HTML and breaks the client.

### Official Anthropic

```env
ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY
# No ANTHROPIC_BASE_URL needed
ANTHROPIC_MODEL=claude-sonnet-4-5
```

### OpenAI / DeepSeek / Ollama (via LiteLLM proxy)

These providers use OpenAI protocol, not Anthropic protocol. Use [LiteLLM](https://github.com/BerriAI/litellm) to translate:

```bash
# Install and run LiteLLM proxy
pip install litellm
litellm --model gpt-4o --port 4000
```

```env
ANTHROPIC_BASE_URL=http://localhost:4000
ANTHROPIC_API_KEY=anything
ANTHROPIC_MODEL=gpt-4o
```

---

## Project Structure

```
bin/claude-haha          # Entry shell script
preload.ts               # Bun preload (MACRO globals)
.env.example             # Env template
src/
├── entrypoints/cli.tsx  # CLI main entry point
├── main.tsx             # TUI logic (Commander.js + React/Ink)
├── localRecoveryCli.ts  # Fallback recovery CLI
├── setup.ts             # Startup initialization
├── screens/REPL.tsx     # Interactive REPL screen
├── ink/                 # Ink terminal rendering engine
├── components/          # UI components
├── tools/               # Agent tools (Bash, Edit, Grep, etc.)
├── commands/            # Slash commands (/commit, /review, etc.)
├── skills/              # Skill system
├── services/            # API, MCP, OAuth services
├── hooks/               # React hooks
└── utils/               # Utility functions
```

---

## Programmatic Usage

You can invoke the CLI entry directly in scripts:

```typescript
// Run headless via Bun shell (example integration)
import { $ } from "bun";

const result = await $`bun --env-file=.env ./src/entrypoints/cli.tsx -p "summarize this file" < myfile.ts`.text();
console.log(result);
```

```typescript
// Check API connectivity before launching
const res = await fetch(`${process.env.ANTHROPIC_BASE_URL}/v1/messages`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.ANTHROPIC_AUTH_TOKEN}`,
    "Content-Type": "application/json",
    "anthropic-version": "2023-06-01",
  },
  body: JSON.stringify({
    model: process.env.ANTHROPIC_MODEL,
    max_tokens: 16,
    messages: [{ role: "user", content: "ping" }],
  }),
});
const data = await res.json();
console.log("API OK:", data.usage?.input_tokens !== undefined);
```

---

## Troubleshooting

### `undefined is not an object (evaluating 'usage.input_tokens')`

**Cause**: `ANTHROPIC_BASE_URL` points to an endpoint that is not returning Anthropic protocol JSON (returns HTML instead).

The SDK appends `/v1/messages` to your base URL automatically. Verify:

```bash
# Should return JSON with usage.input_tokens
curl -X POST "$ANTHROPIC_BASE_URL/v1/messages" \
  -H "Authorization: Bearer $ANTHROPIC_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"'$ANTHROPIC_MODEL'","max_tokens":16,"messages":[{"role":"user","content":"hi"}]}'
```

### `Cannot find package 'bundle'`

```
error: Cannot find package 'bundle' from '.../src/entrypoints/cli.tsx'
```

**Fix**: Upgrade Bun:

```bash
bun upgrade
```

### TUI doesn't start / Enter key unresponsive

The native `modifiers-napi` package may be missing. This is already patched in this repo with a try-catch. If you still hit it, ensure you ran `bun install` cleanly:

```bash
rm -rf node_modules bun.lockb
bun install
```

### Startup hangs / freezes

Check for missing stub files (already created in this repo). If you're seeing hangs on a fresh clone:

```bash
# Verify stub files exist
ls src/skills/verify.md
ls src/ultraplan/prompt.txt
ls src/filePersistence/types.ts
```

If missing, the repo may be incomplete — re-clone or restore from git.

### Falls into Recovery CLI unexpectedly

The original leaked code had `preload.ts` auto-setting `LOCAL_RECOVERY=1`. This repo removes that default. If you're still hitting recovery mode, check:

```bash
echo $LOCAL_RECOVERY          # should be empty
echo $CLAUDE_CODE_FORCE_RECOVERY_CLI   # should be empty
```

Unset them if set:

```bash
unset LOCAL_RECOVERY
unset CLAUDE_CODE_FORCE_RECOVERY_CLI
```

---

## Known Limitations

- **Windows**: Voice input, Computer Use, and Sandbox isolation are unavailable. Core TUI works.
- **Non-Anthropic-protocol providers**: Require LiteLLM or similar proxy.
- **Original source copyright**: All source code copyright Anthropic. This repo is for learning/research only.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | [Bun](https://bun.sh) |
| Language | TypeScript |
| Terminal UI | React + [Ink](https://github.com/vadimdemedes/ink) |
| CLI parsing | Commander.js |
| API client | Anthropic SDK |
| Protocols | MCP, LSP |
```
