```markdown
---
name: t1code-terminal-ui
description: AI-powered terminal coding assistant (T3Code in your terminal) using OpenTUI
triggers:
  - "use t1code in my terminal"
  - "run t1code TUI"
  - "set up terminal AI coding assistant"
  - "t1code configuration"
  - "bunx t1code"
  - "T3Code terminal version"
  - "t1code LLM coding TUI"
  - "install t1code globally"
---

# t1code Terminal UI Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What is t1code?

t1code is a terminal user interface (TUI) for AI-assisted coding, inspired by T3Code (by @t3dotgg and @juliusmarminge). It brings an LLM-powered coding assistant directly into your terminal using the OpenTUI framework. It supports models via API (similar to Codex/Claude/OpenAI-compatible endpoints) and runs entirely in the terminal.

---

## Installation

### Run instantly (no install)

```bash
bunx @maria_rcks/t1code
```

### Install globally

```bash
bun add -g @maria_rcks/t1code
```

After global install, run with:

```bash
t1code
```

### Develop from source

```bash
git clone https://github.com/maria-rcks/t1code.git
cd t1code
bun install
bun dev:tui
```

---

## Requirements

- [Bun](https://bun.sh) runtime (v1.x+)
- A terminal that supports ANSI escape codes
- An LLM API key (OpenAI-compatible endpoint recommended)

---

## Key Commands

| Command | Description |
|---|---|
| `bunx @maria_rcks/t1code` | Run t1code without installing |
| `bun add -g @maria_rcks/t1code` | Install globally |
| `t1code` | Launch TUI (if globally installed) |
| `bun dev:tui` | Run in dev mode from source |
| `bun install` | Install dependencies from source |

---

## Configuration

t1code uses environment variables for API keys and model configuration. Set these before running:

```bash
# OpenAI-compatible API key
export OPENAI_API_KEY=your_api_key_here

# Optional: custom base URL for OpenAI-compatible APIs (e.g. local Ollama, Together, etc.)
export OPENAI_BASE_URL=https://api.openai.com/v1

# Optional: specify default model
export OPENAI_MODEL=gpt-4o
```

You can place these in a `.env` file in the project root when developing from source:

```env
OPENAI_API_KEY=your_api_key_here
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4o
```

---

## TUI Navigation

Once launched, t1code presents a terminal UI with:

- **Chat/prompt input** — Type your coding question or instruction
- **Code output panel** — Displays AI-generated code with syntax highlighting
- **Keyboard shortcuts** — Navigate panels, copy output, clear history

Common TUI interactions:

| Key | Action |
|---|---|
| `Enter` | Submit prompt |
| `Ctrl+C` | Exit t1code |
| `Tab` | Switch focus between panels |
| `Ctrl+L` | Clear the chat/output |
| Arrow keys | Scroll through output |

---

## Development: Project Structure

```
t1code/
├── src/
│   ├── tui/          # OpenTUI components and layout
│   ├── llm/          # LLM API client logic
│   ├── config/       # Configuration loading
│   └── index.ts      # Entry point
├── assets/
│   └── repo/
├── package.json
└── bun.lockb
```

---

## Real Code Examples

### Running t1code programmatically (from source entry point)

```typescript
// src/index.ts - typical entry pattern
import { startTUI } from "./tui";
import { loadConfig } from "./config";

const config = loadConfig();
await startTUI(config);
```

### Extending the LLM client (OpenAI-compatible)

```typescript
// src/llm/client.ts - example OpenAI-compatible fetch
const response = await fetch(`${process.env.OPENAI_BASE_URL}/chat/completions`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
  },
  body: JSON.stringify({
    model: process.env.OPENAI_MODEL ?? "gpt-4o",
    messages: [
      { role: "system", content: "You are a helpful coding assistant." },
      { role: "user", content: userPrompt },
    ],
    stream: true,
  }),
});
```

### Using a local Ollama model with t1code

```bash
# Start Ollama with a code model
ollama run codellama

# Point t1code to local Ollama
export OPENAI_BASE_URL=http://localhost:11434/v1
export OPENAI_API_KEY=ollama
export OPENAI_MODEL=codellama
bunx @maria_rcks/t1code
```

### Using Together AI or other OpenAI-compatible providers

```bash
export OPENAI_API_KEY=$TOGETHER_API_KEY
export OPENAI_BASE_URL=https://api.together.xyz/v1
export OPENAI_MODEL=codellama/CodeLlama-34b-Instruct-hf
bunx @maria_rcks/t1code
```

---

## Common Patterns

### 1. One-off code generation session

```bash
OPENAI_API_KEY=your_key bunx @maria_rcks/t1code
```

### 2. Persistent global install with config in shell profile

```bash
# Add to ~/.bashrc or ~/.zshrc
export OPENAI_API_KEY=your_key
export OPENAI_MODEL=gpt-4o

# Then just run
t1code
```

### 3. Contributing a new TUI component (from source)

```typescript
// src/tui/MyPanel.ts - OpenTUI component pattern
import { Box, Text } from "opentui";

export function MyPanel({ content }: { content: string }) {
  return (
    <Box border="single" padding={1}>
      <Text>{content}</Text>
    </Box>
  );
}
```

### 4. Adding a new keyboard shortcut (from source)

```typescript
// src/tui/keybindings.ts
import { onKey } from "opentui";

onKey("ctrl+r", () => {
  // Reload/reset logic
  resetChat();
});
```

---

## Troubleshooting

### `bunx` not found

Install Bun: https://bun.sh

```bash
curl -fsSL https://bun.sh/install | bash
```

### TUI renders incorrectly / garbled output

- Ensure your terminal supports 256-color ANSI (iTerm2, Warp, kitty, Windows Terminal, etc.)
- Try resizing your terminal window — some TUI layouts require minimum dimensions
- On Windows, use Windows Terminal or WSL2

### API errors / no response

- Verify `OPENAI_API_KEY` is set correctly
- Check that `OPENAI_BASE_URL` points to a valid OpenAI-compatible endpoint
- Test your API key independently:

```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

### `bun dev:tui` fails with module errors

```bash
# Clean install
rm -rf node_modules bun.lockb
bun install
bun dev:tui
```

### Slow streaming / no streaming output

- Confirm your provider supports SSE streaming on the `/chat/completions` endpoint
- Some self-hosted models (e.g., older Ollama builds) may need `stream: false` in the request body

---

## Links

- **npm:** https://www.npmjs.com/package/@maria_rcks/t1code
- **GitHub:** https://github.com/maria-rcks/t1code
- **OpenTUI:** https://github.com/nickvdyck/opentui
- **Original T3Code:** https://github.com/t3dotgg
- **License:** MIT
```
