```markdown
---
name: oh-my-claudecode-orchestration
description: Multi-agent orchestration for Claude Code using Teams, parallel execution, and specialized AI agents
triggers:
  - set up multi-agent claude code
  - orchestrate claude agents in parallel
  - use oh-my-claudecode for automation
  - run multiple claude agents on a task
  - team mode claude code orchestration
  - autopilot build feature with claude
  - configure omc multi-agent workflow
  - parallel ai agent execution claude
---

# oh-my-claudecode Orchestration

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`oh-my-claudecode` (OMC) is a Teams-first multi-agent orchestration layer for Claude Code. It coordinates specialized AI agents in parallel, routes work intelligently by complexity, and runs persistent verify/fix loops — so you describe what to build and it handles the rest.

---

## Installation

### Via Claude Code Plugin Marketplace

```bash
/plugin marketplace add https://github.com/Yeachan-Heo/oh-my-claudecode
/plugin install oh-my-claudecode
```

### Setup

```bash
/setup
/omc-setup
```

### Enable Native Teams (Required for Team Mode)

Add to `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Install CLI via npm

```bash
npm i -g oh-my-claude-sisyphus@latest
# Note: npm package is oh-my-claude-sisyphus, but the tool/commands are oh-my-claudecode
```

---

## Core Concepts

OMC provides multiple orchestration surfaces:

| Mode | Command | Best For |
|------|---------|----------|
| **Team** | `/team N:executor "task"` | Staged pipeline with multiple Claude agents |
| **CLI Workers** | `omc team N:codex "task"` | Codex/Gemini CLI processes in tmux panes |
| **Autopilot** | `autopilot: task` | Single lead agent, end-to-end autonomous |
| **Ralph** | `ralph: task` | Persistent until fully verified complete |
| **Ultrawork** | `ulw task` | Maximum parallelism for burst refactors |
| **CCG** | `/ccg task` | Codex + Gemini + Claude tri-model synthesis |

---

## Team Mode (Canonical — Recommended)

Team runs a staged pipeline: `team-plan → team-prd → team-exec → team-verify → team-fix`

```bash
# Basic team with 3 executor agents
/team 3:executor "fix all TypeScript errors"

# Build a feature with parallel agents
/team 4:executor "implement JWT authentication with refresh tokens"

# Code review via CLI workers
omc team 2:codex "review auth module for security issues"

# UI work with Gemini
omc team 2:gemini "redesign dashboard components for accessibility"

# Mixed team
omc team 1:claude "implement payment flow"
omc team 2:codex "security audit the payment flow"
```

### Team Status & Lifecycle

```bash
# Check status of a running team session
omc team status auth-review

# Shut down a team session
omc team shutdown auth-review
```

---

## Magic Keywords

Use these in natural language prompts — no special syntax required:

```bash
# Autopilot: autonomous end-to-end execution
autopilot: build a REST API for managing tasks with CRUD endpoints

# Ralph: persistent execution with verify/fix loops
ralph: refactor the auth module to use async/await throughout

# Ultrawork: maximum parallelism
ulw fix all ESLint errors across the codebase

# Ralplan: iterative planning with consensus
ralplan this authentication feature

# Deep interview: Socratic requirements clarification
/deep-interview "I want to build a task management app"

# Deepsearch: codebase-focused search
deepsearch for all usages of deprecated crypto functions

# Ultrathink: deep reasoning
ultrathink about the best database schema for this multi-tenant app

# Tri-model synthesis (Codex + Gemini + Claude)
/ccg Review this PR — architecture (Codex) and UI components (Gemini)

# Stop any active OMC mode
stopomc
cancelomc
```

---

## Custom Skills

Skills are reusable knowledge files that auto-inject into context when relevant triggers match.

### Directory Structure

```
.omc/skills/          # Project-scoped (version-controlled, higher priority)
~/.omc/skills/        # User-scoped (applies to all projects)
```

### Skill File Format

```yaml
# .omc/skills/fix-prisma-connection.md
---
name: Fix Prisma Connection Pool
description: Resolves PrismaClientKnownRequestError P2024 connection pool timeout
triggers: ["prisma", "connection pool", "P2024", "timeout", "database"]
source: extracted
---

## Problem
Prisma throws `PrismaClientKnownRequestError` with code `P2024` under high concurrency.

## Solution
Configure connection pool limits in `DATABASE_URL` and adjust `connection_limit`:

```typescript
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  // Append ?connection_limit=5&pool_timeout=30 to DATABASE_URL
}
```

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

Set in `.env`:
```
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=5&pool_timeout=30"
```
```

### Skill Management Commands

```bash
/skill list              # List all loaded skills
/skill add my-skill      # Add a new skill
/skill remove my-skill   # Remove a skill
/skill edit my-skill     # Edit an existing skill
/skill search prisma     # Search skills by keyword
/learner                 # Auto-extract reusable patterns from current session
```

---

## Tri-Model Workflow (CCG)

The `/ccg` skill routes work to Codex (architecture/security) and Gemini (UI/docs), then Claude synthesizes:

```bash
# PR review with specialized models
/ccg Review this PR — check backend logic with Codex and UI consistency with Gemini

# Mixed analysis
/ccg Analyze this codebase: security posture (Codex) and documentation gaps (Gemini)
```

Requires `codex` and `gemini` CLIs installed and an active tmux session.

---

## Updating OMC

### If installed via npm

```bash
npm i -g oh-my-claude-sisyphus@latest
```

### If installed via plugin marketplace

```bash
/plugin marketplace update omc
/omc-setup
```

### Troubleshoot after update

```bash
/omc-doctor
```

---

## Configuration Reference

### `~/.claude/settings.json`

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Project-level skill directory

```bash
mkdir -p .omc/skills
# Add skill files here for team-shared knowledge
```

---

## Real-World Usage Patterns

### Pattern 1: Feature Development Pipeline

```bash
# 1. Clarify requirements first
/deep-interview "multi-tenant SaaS billing system"

# 2. Plan with consensus
ralplan the billing feature based on the interview

# 3. Execute with team
/team 4:executor "implement Stripe billing with subscription tiers per the PRD"
```

### Pattern 2: Codebase Cleanup

```bash
# Parallel TypeScript error fixing
ulw fix all TypeScript compilation errors

# Persistent refactor that won't stop until done
ralph: migrate all class components to React hooks
```

### Pattern 3: Security Audit

```bash
# Codex workers focused on security
omc team 3:codex "audit the entire codebase for OWASP Top 10 vulnerabilities"
omc team status security-audit
```

### Pattern 4: Cross-Model PR Review

```bash
/ccg Review PR #142 — Codex checks algorithmic correctness, Gemini checks API design consistency
```

### Pattern 5: Auto-Learning from Debugging

```bash
# After solving a complex bug
/learner
# OMC extracts the solution as a reusable skill for future sessions
```

---

## Troubleshooting

### Teams not activating

```bash
# Verify settings.json has the env flag
cat ~/.claude/settings.json
# Must contain: "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
```

### OMC falls back to non-team execution

OMC warns and degrades gracefully if teams are disabled. Enable the flag above and restart Claude Code.

### tmux CLI workers fail to spawn

```bash
# Ensure CLIs are installed
which codex    # must resolve
which gemini   # must resolve

# Ensure you're in an active tmux session
echo $TMUX    # must be non-empty
```

### Plugin out of date after install

```bash
/plugin marketplace update omc
/omc-setup
/omc-doctor
```

### Skills not injecting

```bash
/skill list    # verify skill is loaded
/skill search <keyword>  # check trigger words match your prompt
```

### Stop a runaway orchestration

```bash
stopomc
# or
cancelomc
```

---

## Key Notes

- The **npm package** is `oh-my-claude-sisyphus` but the project/commands are `oh-my-claudecode`
- `swarm` keyword was removed in v4.1.7 — migrate to `/team` syntax
- `ralph` automatically includes `ultrawork` parallelism
- Codex/Gemini MCP servers (`x`, `g` providers) were removed in v4.4.0 — use `omc team N:codex` or `omc team N:gemini` instead
- Project skills (`.omc/skills/`) take priority over user skills (`~/.omc/skills/`)
```
