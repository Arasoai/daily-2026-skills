```markdown
---
name: nanoclaw-ai-assistant
description: Lightweight containerized AI assistant built on Anthropic's Agent SDK with multi-channel messaging (WhatsApp, Telegram, Slack, Discord, Gmail), memory, and scheduled jobs
triggers:
  - set up nanoclaw
  - add a messaging channel to nanoclaw
  - create a scheduled task in nanoclaw
  - how do I customize my nanoclaw assistant
  - debug nanoclaw not responding
  - add telegram to nanoclaw
  - configure nanoclaw docker sandboxes
  - nanoclaw agent skills
---

# NanoClaw AI Assistant

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

NanoClaw is a lightweight, containerized AI assistant that runs Claude agents in isolated Linux containers (Docker or Apple Container). It connects to WhatsApp, Telegram, Slack, Discord, and Gmail, supports per-group memory via `CLAUDE.md` files, scheduled jobs, and is built directly on Anthropic's Agent SDK. The entire codebase is intentionally small (~handful of files) so you can read, understand, and customize it completely.

---

## Installation

### Prerequisites

- macOS or Linux
- Node.js 20+
- [Claude Code](https://claude.ai/download) (`claude` CLI)
- Docker Desktop **or** Apple Container (macOS)

### Quick Start

```bash
# Fork and clone
gh repo fork qwibitai/nanoclaw --clone
cd nanoclaw

# Open Claude Code — it handles everything else
claude
```

Then inside the `claude` prompt:

```
/setup
```

Claude Code installs dependencies, authenticates, configures containers, and registers services automatically.

### Docker Sandboxes (Recommended — macOS Apple Silicon)

```bash
curl -fsSL https://nanoclaw.dev/install-docker-sandboxes.sh | bash
```

**Windows (WSL):**

```bash
curl -fsSL https://nanoclaw.dev/install-docker-sandboxes-windows.sh | bash
```

---

## Environment Configuration

NanoClaw uses a `.env` file in the project root. Never commit this file.

```bash
# Required
ANTHROPIC_API_KEY=your_anthropic_api_key

# Optional: use a custom/compatible model endpoint
ANTHROPIC_BASE_URL=https://your-api-endpoint.com
ANTHROPIC_AUTH_TOKEN=your_token_here

# Optional: override the trigger word (default: @Andy)
TRIGGER_WORD=@Andy
```

---

## Architecture Overview

```
Channels --> SQLite --> Polling loop --> Container (Claude Agent SDK) --> Response
```

**Single Node.js process.** Key source files:

| File | Purpose |
|---|---|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel self-registration at startup |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/group-queue.ts` | Per-group queue with global concurrency limit |
| `src/container-runner.ts` | Spawns streaming agent containers |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations (messages, groups, sessions, state) |
| `groups/*/CLAUDE.md` | Per-group persistent memory |

---

## Claude Code Skills (In-Prompt Commands)

These are typed inside the `claude` CLI, **not** your terminal:

```
/setup                        # Full first-time setup
/add-whatsapp                 # Add WhatsApp channel
/add-telegram                 # Add Telegram channel
/add-gmail                    # Add Gmail integration
/customize                    # Guided customization wizard
/convert-to-apple-container   # Switch from Docker to Apple Container (macOS)
/debug                        # Diagnose issues
/clear                        # Compact conversation context (if skill installed)
```

---

## Adding Messaging Channels

Channels self-register at startup when credentials are present. Use skills to add them:

```
# Inside claude prompt:
/add-whatsapp
/add-telegram
/add-gmail
```

Each skill modifies the codebase and adds the necessary credential prompts. After running a skill, credentials are stored in `.env` and the channel activates on next startup.

### Channel File Structure

A channel lives in `src/channels/` and self-registers via the registry:

```typescript
// src/channels/registry.ts — how channels register themselves
import { ChannelRegistry } from './registry';

export interface Channel {
  name: string;
  poll(): Promise<IncomingMessage[]>;
  send(groupId: string, text: string): Promise<void>;
}

// Each channel file calls this at module load time
ChannelRegistry.register(myChannel);
```

Example channel skeleton (e.g. for a custom channel):

```typescript
// src/channels/my-channel.ts
import { ChannelRegistry } from './registry';
import type { Channel, IncomingMessage } from './types';

const myChannel: Channel = {
  name: 'my-channel',

  async poll(): Promise<IncomingMessage[]> {
    // Return new messages since last poll
    // Each message needs: { groupId, senderId, text, timestamp }
    if (!process.env.MY_CHANNEL_TOKEN) return [];
    // ... fetch logic
    return [];
  },

  async send(groupId: string, text: string): Promise<void> {
    // Send text back to the group/user
    // ...
  },
};

ChannelRegistry.register(myChannel);
```

---

## Talking to Your Assistant

Use the trigger word (default `@Andy`) from any connected channel:

```
@Andy what's on my calendar today?
@Andy summarize the last week of git commits in this repo
@Andy send me a Hacker News AI briefing every Monday at 8am
@Andy review docs/README.md and suggest improvements
```

### Main Channel (Self-Chat = Admin)

Your private self-chat is the admin channel. Use it to manage groups and tasks:

```
@Andy list all scheduled tasks
@Andy pause the Monday briefing task
@Andy show me all active groups
@Andy join the "Engineering" Slack group
```

---

## Scheduled Tasks

Tell the assistant to set up recurring jobs in natural language:

```
@Andy every weekday at 9am, send me a sales pipeline summary
@Andy every Friday at 5pm, review git history and update the README if there's drift
@Andy on the 1st of each month, generate an expense report from my receipts folder
```

Tasks are stored in SQLite and managed by `src/task-scheduler.ts`. They run Claude in a container and can message you back with results.

### Viewing and Managing Tasks via Code

```typescript
// src/db.ts — task operations (simplified)
import Database from 'better-sqlite3';
const db = new Database('nanoclaw.db');

// List all scheduled tasks
const tasks = db.prepare('SELECT * FROM scheduled_tasks WHERE active = 1').all();

// Pause a task
db.prepare('UPDATE scheduled_tasks SET active = 0 WHERE id = ?').run(taskId);
```

---

## Per-Group Memory

Each group has its own isolated `CLAUDE.md` at `groups/<group-id>/CLAUDE.md`. The agent reads this file at the start of every conversation in that group.

```markdown
<!-- groups/family-chat/CLAUDE.md -->
# Family Chat Context

- This is the family group. Keep responses warm and casual.
- Dad prefers bullet points.
- Weekly summary every Sunday at 6pm.
- Has access to: /mnt/shared/family-photos
```

To update memory, just tell the assistant:

```
@Andy remember that in this group, always respond in Spanish
@Andy update your memory: this group is for engineering standup, be concise
```

---

## Container Isolation

Every agent invocation spawns a fresh container. Only explicitly mounted directories are accessible — no access to the host filesystem by default.

```typescript
// src/container-runner.ts — simplified spawn logic
import { spawn } from 'child_process';

export async function runAgentInContainer(opts: {
  groupId: string;
  prompt: string;
  mounts: string[];   // e.g. ['~/Documents/work:/mnt/work:ro']
}) {
  const mountArgs = opts.mounts.flatMap(m => ['-v', m]);

  const proc = spawn('docker', [
    'run', '--rm',
    ...mountArgs,
    '-e', `ANTHROPIC_API_KEY=${process.env.ANTHROPIC_API_KEY}`,
    'nanoclaw-agent',
    '--prompt', opts.prompt,
  ]);

  // Stream output back via IPC filesystem watcher
}
```

To give an agent access to a folder, tell it (or modify `groups/<id>/CLAUDE.md`):

```
@Andy you now have access to my Obsidian vault at ~/Documents/Obsidian
```

---

## IPC (Inter-Process Communication)

Agents communicate back to the orchestrator via filesystem IPC watched by `src/ipc.ts`:

```typescript
// src/ipc.ts — watches for agent output files
import chokidar from 'chokidar';

chokidar.watch('ipc/out/*.json').on('add', async (filePath) => {
  const payload = JSON.parse(fs.readFileSync(filePath, 'utf8'));
  await router.send(payload.groupId, payload.text);
  fs.unlinkSync(filePath);
});
```

Agents write responses to `ipc/out/<uuid>.json`:

```json
{
  "groupId": "family-chat",
  "text": "Here's your Monday briefing...",
  "timestamp": "2026-03-17T08:00:00Z"
}
```

---

## Customization Patterns

NanoClaw has no config files by design. Customize by modifying code directly via Claude Code:

```
# Inside claude prompt — natural language changes:
"Change the trigger word from @Andy to @Nova"
"Make responses shorter and more direct by default"
"Add a custom greeting when someone says good morning"
"Store weekly conversation summaries to groups/<id>/summaries/"
"Run /customize for a guided walkthrough"
```

### Example: Changing the Trigger Word Manually

```typescript
// src/index.ts
const TRIGGER_WORD = process.env.TRIGGER_WORD ?? '@Andy';

function isMentioned(text: string): boolean {
  return text.toLowerCase().includes(TRIGGER_WORD.toLowerCase());
}
```

Set in `.env`:
```bash
TRIGGER_WORD=@Nova
```

---

## Contributing a Skill

NanoClaw uses a **skills-over-features** model. Don't add features to core — submit a skill branch.

1. Fork NanoClaw, create a branch `skill/my-feature`
2. Make code changes that implement the feature
3. Add a `SKILL.md` in `skills/add-my-feature/SKILL.md` describing what the skill does
4. Open a PR — maintainers create a `skill/my-feature` branch others can merge

Users install your skill via:
```
/add-my-feature
```

### Skill SKILL.md Structure

```markdown
# /add-signal

Adds Signal as a messaging channel to NanoClaw.

## What this skill does
1. Installs `signal-cli` dependency
2. Creates `src/channels/signal.ts`
3. Registers the channel in the registry
4. Prompts for Signal credentials and writes to `.env`
5. Restarts the orchestrator

## Steps
...
```

---

## Troubleshooting

### Assistant not responding to messages

```
# Inside claude prompt:
/debug
```

Ask Claude Code directly:
- "Why isn't the scheduler running?"
- "What's in the recent logs?"
- "Why did this message not get a response?"

### Container not starting

```bash
# Check Docker is running
docker info

# Test container manually
docker run --rm nanoclaw-agent echo "ok"

# Check logs
docker logs $(docker ps -lq)
```

### Channel not connecting

```bash
# Verify credentials are in .env
cat .env | grep -i channel_name

# Check channel registered at startup
# Look for "Registered channel: <name>" in stdout
npm start 2>&1 | head -50
```

### Database issues

```bash
# Inspect SQLite directly
sqlite3 nanoclaw.db ".tables"
sqlite3 nanoclaw.db "SELECT * FROM scheduled_tasks;"
sqlite3 nanoclaw.db "SELECT * FROM groups;"
```

### Reset a group's memory

```bash
# Edit or delete the group's CLAUDE.md
rm groups/<group-id>/CLAUDE.md
# Or edit it:
nano groups/<group-id>/CLAUDE.md
```

### Switch from Docker to Apple Container (macOS)

```
# Inside claude prompt:
/convert-to-apple-container
```

---

## Using Compatible/Open-Source Models

```bash
# .env — point to any Anthropic API-compatible endpoint
ANTHROPIC_BASE_URL=https://api.together.xyz/v1
ANTHROPIC_AUTH_TOKEN=$TOGETHER_API_KEY

# Ollama (requires an Anthropic-compatible proxy like litellm)
ANTHROPIC_BASE_URL=http://localhost:4000
ANTHROPIC_AUTH_TOKEN=sk-anything
```

---

## Key Links

- **Homepage:** https://nanoclaw.dev
- **Docker Sandboxes announcement:** https://nanoclaw.dev/blog/nanoclaw-docker-sandboxes
- **Security model:** `docs/SECURITY.md`
- **Full spec:** `docs/SPEC.md`
- **Changelog:** `CHANGELOG.md`
- **Discord:** https://discord.gg/VDdww8qS42
```
