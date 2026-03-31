```markdown
---
name: claude-code-source-analysis
description: Expert knowledge of Claude Code v2.1.88 internals, architecture, telemetry, hidden features, and agent loop mechanics extracted from the decompiled npm package source
triggers:
  - "help me understand how Claude Code works internally"
  - "what telemetry does Claude Code collect"
  - "explain the Claude Code agent loop"
  - "what are the hidden features in Claude Code"
  - "how does Claude Code's tool system work"
  - "what is undercover mode in Claude Code"
  - "explain Claude Code's permission system"
  - "how does remote control work in Claude Code"
---

# Claude Code Source Code Analysis

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

This repository contains the unbundled TypeScript source of **Claude Code v2.1.88**, extracted from the `@anthropic-ai/claude-code` npm package. The published package ships a single ~12MB bundled `cli.js`; this repo provides the decompiled `src/` directory for research and analysis.

Key stats:
- ~1,884 TypeScript source files
- ~512,664 lines of code
- 40+ built-in tools
- 80+ slash commands
- Runtime: Bun compiled to Node.js ≥ 18 bundle

---

## Repository Structure

```
src/
├── query.ts               # Core agent loop (~785KB, largest file)
├── tools/                 # 40+ tool implementations
├── commands/              # 80+ slash commands
├── services/              # Telemetry, analytics, state
├── daemon/                # Background daemon (internal only)
├── compact/               # Context compaction strategies
├── skillSearch/           # Skill discovery (internal only)
└── coordinator/           # Multi-agent coordination (internal only)

docs/
├── en/
│   ├── 01-telemetry-and-privacy.md
│   ├── 02-hidden-features-and-codenames.md
│   ├── 03-undercover-mode.md
│   ├── 04-remote-control-and-killswitches.md
│   └── 05-future-roadmap.md
└── zh/                    # Chinese translations of all docs
```

---

## Architecture Overview

### Entry → Query Engine → Tools/Services/State

```
User Input
    │
    ▼
messages[] array
    │
    ▼
┌─────────────────────────────────────┐
│           query.ts                  │
│  (Core Agent Loop / Query Engine)   │
│                                     │
│  1. Build system prompt             │
│  2. Apply harness mechanisms        │
│  3. Send to Claude API              │
│  4. Parse tool_use blocks           │
│  5. Execute tools                   │
│  6. Append tool_result              │
│  7. Loop until stop_reason=end_turn │
└─────────────────────────────────────┘
    │
    ▼
Tool Execution Layer
├── File tools (Read, Write, Edit)
├── Shell tools (Bash, Execute)
├── Search tools (Grep, Find)
├── Web tools (WebFetch, WebSearch)
└── Agent tools (Task, SubAgent)
```

### The 12 Progressive Harness Mechanisms

Claude Code layers these mechanisms on the core agent loop:

| # | Mechanism | Purpose |
|---|-----------|---------|
| 1 | **Prompt Injection Guard** | Detects adversarial content in tool results |
| 2 | **Context Window Manager** | Tracks token usage, triggers compaction |
| 3 | **Permission Gate** | Validates tool calls against user permissions |
| 4 | **Telemetry Hooks** | Fires analytics events on every action |
| 5 | **Feature Flag Resolver** | GrowthBook-based runtime feature switching |
| 6 | **Remote Settings Poller** | Hourly `/api/claude_code/settings` check |
| 7 | **Undercover Mode Filter** | Strips AI attribution in public repos |
| 8 | **Effort Anchor** | Internal users get enhanced reasoning prompts |
| 9 | **Verification Agent** | Plan verification before execution (internal) |
| 10 | **Compact Strategies** | Snip/reactive/micro-compact for long sessions |
| 11 | **Sub-Agent Spawner** | Parallel task execution via Task tool |
| 12 | **Kill Switch Enforcer** | Blocks features per remote killswitch state |

---

## Tool System Architecture

### Available Tools (Public, ~40+)

```typescript
// Tools are registered with a permission level
interface Tool {
  name: string;
  description: string;
  input_schema: JSONSchema;
  permissionLevel: 'read' | 'write' | 'execute' | 'network';
  needsConfirmation: boolean;
}

// Example: Reading a file (no confirmation needed)
{
  name: "Read",
  permissionLevel: "read",
  needsConfirmation: false
}

// Example: Running bash (confirmation required unless --dangerously-skip-permissions)
{
  name: "Bash",
  permissionLevel: "execute",
  needsConfirmation: true
}
```

### Permission Flow

```
Tool Call Requested
        │
        ▼
Is tool in allowlist?
   ├─ YES → Execute immediately
   └─ NO  → Check permission level
                │
                ▼
         Show confirmation dialog
                │
         User approves?
         ├─ YES → Execute + add to session allowlist
         └─ NO  → Return permission denied to model
```

### Sub-Agent Pattern

```typescript
// Task tool spawns isolated sub-agents
const taskResult = await Task({
  description: "Implement the authentication module",
  prompt: "Create src/auth/index.ts with JWT support...",
  // Sub-agent gets its own context window
  // Results returned as text to parent agent
});
```

---

## Telemetry & Privacy

### What's Collected

Two analytics sinks fire on every meaningful event:

```
Event → Anthropic 1st-party logging (no UI opt-out)
      → Datadog (opt-outable via env var)
```

Every event includes:
- Environment fingerprint (OS, shell, terminal)
- Process metrics (memory, CPU)
- Repository hash (SHA of repo path)
- Session ID and conversation ID
- Model used and token counts

### Enable Full Tool Input Capture

```bash
# WARNING: This logs ALL tool inputs including file contents
export OTEL_LOG_TOOL_DETAILS=1
claude
```

### Disable Datadog Analytics

```bash
export CLAUDE_CODE_DISABLE_DATADOG=1
```

> **Note**: First-party Anthropic logging has no UI-exposed opt-out mechanism.

---

## Hidden Features & Codenames

### Model Codenames

| Codename | Model | Status |
|----------|-------|--------|
| Capybara v8 | Claude 3.x | Current |
| Tengu | Claude 3.5 Sonnet | Current |
| Fennec | Opus 4.6 | Near-term |
| **Numbat** | Next generation | In development |

### Feature Flags

Feature flags use random word pairs to obscure their purpose:

```typescript
// Flags are checked at runtime via GrowthBook
feature('tengu_frond_boric')  // → enables some Tengu-specific behavior
feature('KAIROS')              // → fully autonomous agent mode
feature('DAEMON')              // → background daemon supervisor
feature('PROACTIVE')           // → proactive notifications
```

### Hidden Slash Commands

```bash
/btw      # Internal communication channel
/stickers # Sticker/reaction system
```

---

## Undercover Mode

When Anthropic employees work in **public open-source repositories**, the system automatically enables undercover mode:

```
Trigger: Anthropic email domain + public repo detected
Effect:  Model instructed to hide AI authorship

System prompt addition:
  "Do not blow your cover. Write commits and comments
   as a human developer would. Strip all AI attribution."
```

> **There is no force-OFF switch for undercover mode.**

---

## Remote Control & Killswitches

### Settings Polling

```
Every 60 minutes:
  GET /api/claude_code/settings
    │
    ▼
  Parse managed settings
    │
    ▼
  Dangerous change detected?
  ├─ YES → Show blocking dialog
  │          User rejects?
  │          └─ App exits immediately
  └─ NO  → Apply silently
```

### Known Killswitches (6+)

```typescript
killswitch('bypass_permissions')   // Force-enable dangerous mode
killswitch('fast_mode')            // Skip confirmation dialogs
killswitch('voice_mode')           // Enable/disable push-to-talk
killswitch('analytics_sink')       // Switch telemetry destination
killswitch('model_override')       // Force specific model version
killswitch('feature_flags_reset')  // Wipe all GrowthBook overrides
```

---

## Future Roadmap (Found in Source)

### KAIROS — Autonomous Agent Mode

```typescript
// KAIROS adds <tick> heartbeats to the agent loop
// Enables push-based task management
interface KAIROSConfig {
  tickIntervalMs: number;        // Heartbeat interval
  prSubscriptions: string[];     // GitHub PRs to monitor
  pushNotifications: boolean;    // Notify on task completion
  backgroundMode: boolean;       // Run without user present
}
```

### 17 Unreleased Tools (DCE'd from bundle)

```typescript
REPLTool              // Interactive VM sandbox
SnipTool              // Context snipping
SleepTool             // Delay in agent loop
MonitorTool           // MCP monitoring
WebBrowserTool        // Browser automation
TerminalCaptureTool   // Terminal capture
VerifyPlanExecutionTool // Plan verification
SendUserFileTool      // Send files to users
SubscribePRTool       // GitHub PR subscriptions
SuggestBackgroundPRTool // Background PR suggestions
PushNotificationTool  // Push notifications
CtxInspectTool        // Context inspection
ListPeersTool         // List active peers
DiscoverSkillsTool    // Skill discovery
WorkflowTool          // Workflow execution
TungstenTool          // Internal perf monitoring
OverflowTestTool      // Overflow testing
```

### Voice Mode

Push-to-talk voice input is implemented and gated behind a killswitch. Not yet enabled for any users.

---

## Missing Modules (108 Total)

This source is **incomplete**. 108 modules exist only in Anthropic's internal monorepo:

```
Anthropic Internal Build          Published npm Package
────────────────────────          ─────────────────────
feature('DAEMON') → true    →     feature('DAEMON') → false
daemon/main.js    INCLUDED  →     daemon/main.js    DELETED (DCE)
tools/REPLTool    INCLUDED  →     tools/REPLTool    DELETED (DCE)
```

Categories of missing modules:
- **~70 internal modules**: daemon, coordinator, KAIROS assistant, skill search
- **~20 feature-gated tools**: REPL, browser, workflow, notification tools
- **~6 prompt assets**: Internal system prompts and classifier templates

---

## Working With This Source

### Browsing the Source

```bash
git clone https://github.com/sanbuphy/claude-code-source-code
cd claude-code-source-code

# Largest and most important file
wc -l src/query.ts  # ~785KB, the entire agent loop

# Tool implementations
ls src/tools/

# Slash commands
ls src/commands/
```

### Searching for Specific Behaviors

```bash
# Find telemetry events
grep -r "track(" src/ --include="*.ts" | head -20

# Find permission checks
grep -r "needsConfirmation" src/ --include="*.ts"

# Find feature flags
grep -r "feature(" src/ --include="*.ts" | grep -v "node_modules"

# Find killswitch checks
grep -r "killswitch(" src/ --include="*.ts"
```

### Why This Source Won't Compile

```
1. 108 imported modules don't exist in this repo
2. Bun-specific APIs (feature(), Bun.build()) used throughout
3. Internal type declarations missing (coreTypes.generated.js)
4. Protected namespaces reference Anthropic-internal packages
```

---

## Reading the Deep Analysis Reports

```bash
# Start with telemetry analysis
cat docs/en/01-telemetry-and-privacy.md

# Understand hidden features
cat docs/en/02-hidden-features-and-codenames.md

# Undercover mode details
cat docs/en/03-undercover-mode.md

# Remote control mechanisms
cat docs/en/04-remote-control-and-killswitches.md

# Future roadmap
cat docs/en/05-future-roadmap.md
```

---

## Common Research Patterns

### Trace a Tool Call End-to-End

```bash
# 1. Find tool registration
grep -r "registerTool\|ToolRegistry" src/ --include="*.ts"

# 2. Find permission check
grep -r "checkPermission\|permissionLevel" src/tools/ --include="*.ts"

# 3. Find telemetry emission
grep -r "tool_use\|tool_result" src/services/ --include="*.ts"
```

### Understand the Context Window Management

```bash
# Find compaction triggers
grep -r "compact\|compaction\|contextWindow" src/ --include="*.ts"

# Find token counting
grep -r "countTokens\|tokenCount" src/ --include="*.ts"
```

### Analyze the System Prompt Construction

```bash
# System prompt is built in query.ts
grep -n "systemPrompt\|buildPrompt\|SYSTEM" src/query.ts | head -30
```

---

## Troubleshooting Research

**Q: I can't import the source files**
> This source requires Bun's runtime APIs and 108 missing internal modules. It is for analysis only, not execution.

**Q: Some modules are missing**
> 108 modules were dead-code-eliminated from the bundle. See the Missing Modules section — they cannot be recovered from any published artifact.

**Q: Feature flags return false**
> `feature()` is a Bun compile-time intrinsic. In this extracted source it always returns the external (false) value since the internal build flags aren't set.

**Q: How do I find the system prompt?**
> Search `src/query.ts` for `systemPrompt` — it's constructed dynamically based on context, feature flags, and whether the user is an Anthropic employee.
```
