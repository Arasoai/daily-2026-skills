```markdown
---
name: claude-code-source-analysis
description: Skill for analyzing, navigating, and understanding the Claude Code v2.1.88 decompiled TypeScript source, its architecture, tools, feature flags, and internal systems
triggers:
  - analyze claude code source
  - explore claude code internals
  - understand claude code architecture
  - claude code tool system
  - claude code feature flags
  - navigate claude code source
  - claude code telemetry analysis
  - claude code agent loop internals
---

# Claude Code Source Code Analysis

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

This repository contains the **unbundled TypeScript source** extracted from the `@anthropic-ai/claude-code` npm package (v2.1.88). The published package ships a single ~12MB `cli.js` bundle; this repo reverse-engineers and presents the internal module structure for research and educational purposes. It covers the query engine, 40+ tools, permission model, telemetry systems, feature flags, remote control mechanisms, and future roadmap codenames.

---

## What This Repository Is

- **Source**: Extracted from `@anthropic-ai/claude-code@2.1.88` on npm
- **Language**: TypeScript (unbundled from `cli.js`)
- **Purpose**: Technical research, education, architecture study
- **License**: Intellectual property of Anthropic — no commercial use
- **Incomplete**: 108 modules are missing (internal Anthropic monorepo only)

---

## Repository Structure

```
claude-code-source-code/
├── src/                  # Unbundled TypeScript source modules
├── docs/
│   ├── en/               # English deep-analysis reports
│   ├── ja/               # 日本語
│   ├── ko/               # 한국어
│   └── zh/               # 中文
├── README.md
├── README_CN.md
├── README_KR.md
└── README_JA.md
```

---

## Key Analysis Reports (`docs/en/`)

| File | Topic |
|------|-------|
| `01-telemetry-and-privacy.md` | What data is collected, Datadog + 1P sinks, no opt-out |
| `02-hidden-features-and-codenames.md` | Capybara/Tengu/Numbat codenames, feature flags, hidden commands |
| `03-undercover-mode.md` | AI authorship hiding in open-source repos |
| `04-remote-control-and-killswitches.md` | Hourly settings polling, killswitches, GrowthBook flags |
| `05-future-roadmap.md` | KAIROS autonomous mode, voice mode, 17 unreleased tools |

---

## Architecture Overview

### Entry → Query Engine → Tools/Services/State

```
CLI Entry (cli.js)
  └── Query Engine
        ├── Agent Loop  (message → tool call → result → repeat)
        ├── Tool System (40+ registered tools)
        ├── Permission Manager
        ├── State / Session
        └── Services
              ├── Telemetry (Anthropic 1P + Datadog)
              ├── Remote Settings (hourly poll)
              └── Feature Flags (GrowthBook)
```

### The 12 Progressive Harness Mechanisms

Claude Code layers production features on top of the base agent loop:

1. **System prompt injection** — role, context, repo hash
2. **Tool registration** — dynamic based on feature flags
3. **Permission gating** — user approval flow per tool
4. **Telemetry hooks** — event on every tool call/result
5. **Remote settings overlay** — managed org policies
6. **Feature flag evaluation** — GrowthBook per-user
7. **Undercover mode** — strips AI attribution in public repos
8. **Sub-agent spawning** — coordinator/worker pattern
9. **Context compaction** — sliding window + micro-compact
10. **Killswitch enforcement** — remote disable of capabilities
11. **Model override** — remote can change model mid-session
12. **Effort anchors** — internal users get higher quality prompts

---

## Tool System Architecture

### Published Tools (40+)

Claude Code registers tools dynamically. Core published tools include:

```typescript
// Tool registration pattern (from src/)
interface Tool {
  name: string;
  description: string;
  input_schema: JSONSchema;
  execute(input: unknown, context: ToolContext): Promise<ToolResult>;
}

// Example: BashTool
const BashTool: Tool = {
  name: "Bash",
  description: "Execute shell commands in the current working directory",
  input_schema: {
    type: "object",
    properties: {
      command: { type: "string" },
      timeout: { type: "number" }
    },
    required: ["command"]
  },
  async execute({ command, timeout }, ctx) {
    // Permission check → exec → return stdout/stderr
  }
};
```

### Feature-Gated Tools (not in npm bundle)

```typescript
// These exist in sdk-tools.d.ts type signatures but are DCE'd:
// REPLTool         — interactive VM sandbox (internal)
// WebBrowserTool   — browser automation (WEB_BROWSER_TOOL flag)
// SleepTool        — agent loop delay (KAIROS/PROACTIVE)
// PushNotificationTool — push notifs (KAIROS)
// SubscribePRTool  — GitHub PR webhooks (KAIROS_GITHUB_WEBHOOKS)
// VerifyPlanExecutionTool — plan verification
// DiscoverSkillsTool — skill discovery (EXPERIMENTAL_SKILL_SEARCH)
```

### Permission Flow

```typescript
// Simplified permission model
type PermissionLevel = "always" | "ask" | "never";

interface PermissionRequest {
  tool: string;
  input: unknown;
  riskLevel: "low" | "medium" | "high";
}

// Flow: Tool.execute() → PermissionManager.check() → 
//   if "ask": show UI prompt → user approves/denies
//   if "always": proceed
//   if "never": throw PermissionDeniedError
```

---

## Telemetry System

### Two Analytics Sinks

```typescript
// 1st-party sink — Anthropic internal
// 2nd-party sink — Datadog

// Every event includes:
interface TelemetryEvent {
  event_name: string;
  session_id: string;
  repo_hash: string;         // hashed git remote URL
  environment_fingerprint: string;
  process_metrics: ProcessMetrics;
  timestamp: number;
  // ...tool-specific fields
}

// Enable full tool input capture (research/debug):
// OTEL_LOG_TOOL_DETAILS=1 claude ...
```

### No UI Opt-Out for 1P Logging

The `DISABLE_TELEMETRY` / `CLAUDE_CODE_DISABLE_NONESSENTIAL_MEMORY` env vars affect some sinks but **not** the first-party Anthropic analytics stream. This is documented in `docs/en/01-telemetry-and-privacy.md`.

---

## Feature Flags

### GrowthBook Integration

```typescript
// Feature flags use obfuscated random-word-pair names to hide purpose:
// "tengu_frond_boric"   → enables Tengu model routing
// "capybara_v8_stable"  → Capybara v8 rollout

// Evaluation happens at startup + on remote settings refresh
const flagValue = growthbook.isOn("tengu_frond_boric");
if (flagValue) {
  // enable Tengu-specific behavior
}
```

### Known Codenames

| Codename | Maps To | Status |
|----------|---------|--------|
| Capybara v8 | Claude model series | Stable |
| Tengu | Model variant | Rollout |
| Fennec | Opus 4.6 | Released |
| Numbat | Next model | In development |
| KAIROS | Autonomous agent mode | Gated |

---

## Remote Control & Killswitches

### Hourly Settings Poll

```typescript
// Claude Code polls every ~60 minutes:
// GET /api/claude_code/settings

interface ManagedSettings {
  model_override?: string;       // force a specific model
  killswitches: KillswitchSet;
  org_policies: PolicyMap;
}

interface KillswitchSet {
  bypass_permissions?: boolean;
  fast_mode?: boolean;
  voice_mode?: boolean;
  analytics_sink?: boolean;
  // 2+ more undocumented
}
```

### Dangerous Setting Changes

If a remote settings update includes a "dangerous" change, a **blocking dialog** appears. If the user rejects it, **the application exits**. This is documented in `docs/en/04-remote-control-and-killswitches.md`.

---

## Undercover Mode

```typescript
// Triggered automatically for Anthropic employees in public repos.
// System prompt addition (paraphrased from decompiled source):
const UNDERCOVER_PROMPT = `
  You are in undercover mode. Do not reveal that you are an AI or Claude.
  Do not blow your cover. Write commits and code comments as a human 
  developer would. Strip all AI attribution from outputs.
`;

// No force-OFF mechanism exists in the published codebase.
// Raises transparency concerns for open-source communities.
```

---

## KAIROS — Autonomous Agent Mode

KAIROS is the next-gen autonomous mode found in feature-gated code:

```typescript
// KAIROS uses <tick> heartbeat messages for long-running sessions
// Agent loop receives periodic ticks and can:
//   - Send push notifications (PushNotificationTool)
//   - Subscribe to GitHub PR events (SubscribePRTool)
//   - Suggest background PR creation (SuggestBackgroundPRTool)
//   - Send files to users (SendUserFileTool)

// Voice mode (push-to-talk) is implemented but gated:
// Killswitch: voice_mode = false (default in production)
```

---

## Hidden Commands

Discovered in the decompiled source:

```bash
/btw        # undocumented internal command
/stickers   # undocumented internal command

# These appear in internal user builds; behavior in public builds unknown
```

---

## Missing Modules (108 total)

### Internal Anthropic Infrastructure (~70 modules)

These cannot be recovered from any published artifact:

```
daemon/main.js                    # background daemon supervisor
daemon/workerRegistry.js          # daemon worker registry
proactive/index.js                # proactive notifications
contextCollapse/index.js          # context collapse (experimental)
skillSearch/remoteSkillLoader.js  # remote skill loading
coordinator/workerAgent.js        # multi-agent coordinator
assistant/index.js                # KAIROS assistant mode
bridge/peerSessions.js            # bridge peer sessions
commands/subscribe-pr.js          # GitHub PR subscription
commands/workflows/index.js       # workflow commands
```

### Checking Feature Gates

```typescript
// Pattern used throughout the codebase to guard missing modules:
import { feature } from "./featureFlags";

if (feature("KAIROS")) {
  const { assistant } = await import("./assistant/index.js");
  // This import fails in the npm package — module not present
}
```

---

## Environment Variables Reference

```bash
# Telemetry
OTEL_LOG_TOOL_DETAILS=1          # capture full tool inputs in telemetry
DISABLE_TELEMETRY=1              # disables SOME sinks (not 1P Anthropic)
CLAUDE_CODE_DISABLE_NONESSENTIAL_MEMORY=1  # reduces memory telemetry

# Debug / Research
CLAUDE_CODE_DEBUG=1              # verbose logging
ANTHROPIC_LOG=debug              # SDK-level debug logs

# API
ANTHROPIC_API_KEY=$YOUR_KEY      # standard API key
ANTHROPIC_BASE_URL=https://...   # custom base URL (proxy, etc.)

# Feature flags (internal)
CLAUDE_CODE_ENABLE_FEATURE=...   # override specific feature gates
```

---

## Navigating the Source

```bash
# Clone the repository
git clone https://github.com/sanbuphy/claude-code-source-code
cd claude-code-source-code

# Read analysis reports
cat docs/en/01-telemetry-and-privacy.md
cat docs/en/05-future-roadmap.md

# Browse source modules
ls src/

# Search for specific patterns
grep -r "KAIROS" src/
grep -r "undercover" src/
grep -r "killswitch" src/
grep -r "feature(" src/ | head -40
```

> **Note**: The `src/` directory is **not directly compilable** — 108 modules are missing, type references are incomplete, and the build system relies on Anthropic's internal monorepo infrastructure.

---

## Troubleshooting

### "Cannot find module" errors when trying to build

Expected — 108 internal modules are missing. This source is for **analysis only**, not compilation.

### Feature flag behavior differs from docs

GrowthBook flags are evaluated per-user and can change remotely. Internal Anthropic users get different flag states than external users.

### Telemetry still fires with DISABLE_TELEMETRY=1

Correct per the analysis — the 1st-party Anthropic sink bypasses the standard opt-out mechanism. See `docs/en/01-telemetry-and-privacy.md` for full details.

### Undercover mode active unexpectedly

Undercover mode auto-activates for Anthropic employee accounts in detected public repos. No toggle exists in the published codebase.

---

## Related Resources

- [npm package](https://www.npmjs.com/package/@anthropic-ai/claude-code) — `@anthropic-ai/claude-code`
- [Anthropic Docs](https://docs.anthropic.com/claude/docs/claude-code) — Official Claude Code documentation
- [Analysis thread](https://x.com/Fried_rice/status/2038894956459290963) — Original discovery thread
- `docs/` directory — Full quadrilingual analysis reports
```
