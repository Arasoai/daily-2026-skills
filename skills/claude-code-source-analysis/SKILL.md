```markdown
---
name: claude-code-source-analysis
description: Expertise in the Claude Code v2.1.88 decompiled TypeScript source — architecture, tool system, telemetry, feature flags, hidden modules, and internal mechanisms
triggers:
  - analyze claude code source
  - claude code internals
  - claude code architecture
  - claude code tool system
  - claude code telemetry privacy
  - claude code feature flags
  - claude code hidden features
  - explore claude code source code
---

# Claude Code Source Code Analysis

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

This repository contains the decompiled and unbundled TypeScript source extracted from the `@anthropic-ai/claude-code` npm package (v2.1.88). The published package ships as a single ~12MB `cli.js` bundle; this repo reconstructs the `src/` directory for research and educational study. It includes deep analysis reports on telemetry, hidden features, undercover mode, remote control mechanisms, and future roadmap items.

---

## Project Structure

```
/
├── src/                  # Unbundled TypeScript source (extracted from cli.js)
├── docs/
│   ├── en/               # English analysis reports
│   ├── ja/               # Japanese reports
│   ├── ko/               # Korean reports
│   └── zh/               # Chinese reports
├── README.md
└── README_CN.md / README_KR.md / README_JA.md
```

---

## Key Analysis Reports (`docs/en/`)

| File | Topic |
|------|-------|
| `01-telemetry-and-privacy.md` | What is collected, two analytics sinks, no UI opt-out |
| `02-hidden-features-and-codenames.md` | Animal codenames, feature flags, hidden commands |
| `03-undercover-mode.md` | AI authorship concealment in open-source repos |
| `04-remote-control-and-killswitches.md` | Hourly settings polling, killswitches, GrowthBook flags |
| `05-future-roadmap.md` | KAIROS, Numbat, voice mode, 17 unreleased tools |

---

## Architecture Overview

Claude Code follows an **Entry → Query Engine → Tools/Services/State** pipeline:

```
CLI Entry (cli.js)
  └─► Query Engine (agent loop)
        ├─► Tool System (40+ tools)
        ├─► Services (telemetry, settings, session)
        └─► State (conversation history, context)
```

### Agent Loop (simplified reconstruction)

```typescript
// Reconstructed from src/ analysis
async function agentLoop(userMessage: string, context: AgentContext) {
  while (true) {
    const response = await callClaude(context.messages);

    if (response.stopReason === 'end_turn') break;

    if (response.stopReason === 'tool_use') {
      const toolResults = await executeTools(response.toolUses, context);
      context.messages.push(...toolResults);
      continue;
    }

    break;
  }
  return response;
}
```

---

## Tool System Architecture

Claude Code exposes **40+ tools** to the agent. They are split into categories:

### Published Tools (in npm bundle)

```typescript
// Core file tools
BashTool          // Shell command execution
ReadFileTool      // Read file contents
WriteFileTool     // Write/overwrite files
EditFileTool      // Patch-style edits
ListFilesTool     // Directory listing
GlobTool          // Pattern-based file search
GrepTool          // Regex search in files

// Agent tools
AgentTool         // Spawn sub-agents
TaskTool          // Task management
TodoReadTool      // Read todo list
TodoWriteTool     // Write todo list

// Web / external
WebFetchTool      // HTTP fetch
WebSearchTool     // Web search (when enabled)

// MCP
MCPTool           // Model Context Protocol integration
```

### Feature-Gated Tools (stripped from bundle, type stubs only)

```typescript
// These exist in sdk-tools.d.ts but not in cli.js
WebBrowserTool        // Browser automation     — WEB_BROWSER_TOOL flag
REPLTool              // VM sandbox REPL         — internal (ant)
SleepTool             // Agent loop delay        — PROACTIVE / KAIROS
PushNotificationTool  // Push notifications      — KAIROS
SubscribePRTool       // GitHub PR subscription  — KAIROS_GITHUB_WEBHOOKS
WorkflowTool          // Workflow execution      — WORKFLOW_SCRIPTS
MonitorTool           // MCP monitoring          — MONITOR_TOOL
SnipTool              // Context snipping        — HISTORY_SNIP
```

---

## Telemetry & Privacy

### Two Analytics Sinks

```
User Action
  ├─► 1st-party sink  → Anthropic servers  (no UI opt-out)
  └─► Datadog          → metrics/monitoring (configurable)
```

### What Is Collected Per Event

```typescript
interface TelemetryEvent {
  event_name: string;
  session_id: string;
  environment_fingerprint: string;  // OS, node version, shell
  process_metrics: ProcessMetrics;  // CPU, memory
  repo_hash: string;                // SHA of git remote URL
  timestamp: number;
  // ... additional fields
}
```

### Enabling Full Tool Input Capture

```bash
# WARNING: captures full tool inputs including file contents
export OTEL_LOG_TOOL_DETAILS=1
claude
```

### Partial Opt-Out (Datadog only)

```bash
# In settings or environment — disables Datadog sink only
CLAUDE_CODE_DISABLE_TELEMETRY=1
```

---

## Feature Flags

Feature flags use **random word-pair names** to obscure their purpose (e.g., `tengu_frond_boric`). They are fetched from GrowthBook and can change behavior remotely without user consent.

```typescript
// Pattern used throughout src/
if (feature('KAIROS')) {
  // load kairos assistant mode
  await import('./assistant/index.js');
}

if (feature('DAEMON')) {
  await import('./daemon/main.js');
}

if (feature('WEB_BROWSER_TOOL')) {
  tools.push(new WebBrowserTool());
}
```

### Known Feature Flags

| Flag | Purpose |
|------|---------|
| `KAIROS` | Fully autonomous agent mode with `<tick>` heartbeats |
| `DAEMON` | Background daemon supervisor |
| `PROACTIVE` | Proactive notifications |
| `COORDINATOR_MODE` | Multi-agent coordination |
| `BRIDGE_MODE` | Peer session bridging |
| `HISTORY_SNIP` | Context snip-based compaction |
| `CACHED_MICROCOMPACT` | Reactive context compaction |
| `EXPERIMENTAL_SKILL_SEARCH` | Remote skill discovery |
| `WORKFLOW_SCRIPTS` | Workflow execution |
| `KAIROS_GITHUB_WEBHOOKS` | PR subscription |
| `WEB_BROWSER_TOOL` | Browser automation |
| `BUDDY` | Buddy system notifications |
| `FORK_SUBAGENT` | Fork sub-agent command |
| `AUTO_THEME` | System theme watcher |
| `UDS_INBOX` | Unix domain socket messaging |

---

## Remote Control Mechanism

Claude Code polls `https://api.anthropic.com/api/claude_code/settings` **every hour**.

```typescript
// Reconstructed polling logic
async function pollRemoteSettings() {
  const settings = await fetch('/api/claude_code/settings');

  if (settings.dangerous_change) {
    const accepted = await showBlockingDialog(settings.change_description);
    if (!accepted) {
      process.exit(1);  // Reject = app exits
    }
  }

  applySettings(settings);
}

setInterval(pollRemoteSettings, 60 * 60 * 1000);
```

### Known Killswitches (remotely togglable)

- Bypass permissions
- Fast mode toggle
- Voice mode enable/disable
- Analytics sink routing
- Model override (force specific model)
- Feature flag overrides via GrowthBook

---

## Undercover Mode

When an **Anthropic employee** authenticates and works in a **public open-source repository**, undercover mode activates automatically:

```typescript
// Reconstructed from decompiled source
if (isAnthropicEmployee(user) && isPublicRepo(repoContext)) {
  systemPrompt += `
    You are in undercover mode.
    Do not blow your cover.
    Strip all AI attribution from commits and code.
    Write commits as a human developer would.
    Do not mention Claude, Anthropic, or AI assistance.
  `;
}
```

**No force-OFF mechanism exists** in the published code.

---

## Model Codenames

| Codename | Model |
|----------|-------|
| Capybara v8 | Claude 3.x (current) |
| Tengu | Internal variant |
| Fennec | Opus 4.6 |
| **Numbat** | Next-gen (unreleased) |

---

## KAIROS — Autonomous Agent Mode

KAIROS is a fully autonomous background-agent architecture found in feature-gated code:

```typescript
// Reconstructed KAIROS tick loop (not in npm bundle)
async function kairosTick(session: KairosSession) {
  // Heartbeat-driven autonomous loop
  while (session.active) {
    await processInbox(session);           // Check UDS inbox
    await processPRSubscriptions(session); // GitHub webhook events
    const tick = await generateTick(session);
    await executeTick(tick);
    await sleep(session.tickInterval);
  }
}
```

Unreleased KAIROS tools: `PushNotificationTool`, `SubscribePRTool`, `SuggestBackgroundPRTool`, `SleepTool`, `SendUserFileTool`.

---

## Hidden Commands

| Command | Purpose |
|---------|---------|
| `/btw` | Internal side-channel command |
| `/stickers` | Internal sticker/easter egg |
| `/fork` | Fork sub-agent (FORK_SUBAGENT flag) |
| `/peers` | List active peers (BRIDGE_MODE flag) |
| `/subscribe-pr` | GitHub PR subscription (KAIROS_GITHUB_WEBHOOKS) |

---

## 108 Missing Modules

108 modules are referenced in the source but absent from the npm package (dead-code-eliminated or internal-only). They **cannot be recovered** from `cli.js`. Examples:

```
daemon/main.js                  — Background daemon
proactive/index.js              — Proactive notifications
contextCollapse/index.js        — Experimental context collapse
skillSearch/remoteSkillLoader.js — Remote skill loader
coordinator/workerAgent.js      — Multi-agent coordinator
assistant/index.js              — KAIROS assistant mode
bridge/peerSessions.js          — Peer session management
commands/workflows/index.js     — Workflow commands
```

---

## Build Notes

This source is **not directly compilable** because:

1. 108 internal modules are absent
2. Generated files (`coreTypes.generated.js`) are missing
3. Internal Anthropic infrastructure (`protectedNamespace.js`, `devtools.js`) is not published
4. Some imports reference private npm scopes (`@anthropic-internal/*`)

To study the code, use it as a **read-only reference** alongside the published `cli.js` bundle.

---

## Exploring the Source

```bash
# Clone the repo
git clone https://github.com/sanbuphy/claude-code-source-code.git
cd claude-code-source-code

# Read architecture overview
cat README.md

# Read telemetry report
cat docs/en/01-telemetry-and-privacy.md

# Browse source
ls src/

# Cross-reference with published bundle (requires npm install)
npm install -g @anthropic-ai/claude-code@2.1.88
# Bundle location:
node -e "console.log(require.resolve('@anthropic-ai/claude-code'))"
```

---

## Ethical & Legal Notes

- All source is **Anthropic's intellectual property**
- This repo is for **research and education only**
- **Commercial use is prohibited**
- If you believe content infringes your rights, contact the repo owner for removal
- The undercover mode and remote control findings raise **open-source transparency concerns** worth public discussion
```
