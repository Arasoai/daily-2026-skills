```markdown
---
name: citadel-agent-orchestration
description: Agent orchestration harness for Claude Code with four-tier routing, campaign persistence, parallel fleet agents, and 25 skills for autonomous coding campaigns.
triggers:
  - set up citadel for my project
  - run parallel agents with citadel
  - orchestrate a coding campaign
  - use citadel to build a feature
  - how do I use the /do router
  - run a fleet of agents in parallel
  - persist a coding campaign across sessions
  - automate code review with citadel
---

# Citadel — Agent Orchestration Harness for Claude Code

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Citadel is a Claude Code **plugin** that routes any coding task through the cheapest capable tool — from a direct one-line fix to a multi-day parallel agent campaign. It provides 25 skills, 4 autonomous sub-agents, 10 lifecycle hooks, and campaign persistence across sessions.

---

## Installation

**Prerequisites:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code) + Node.js 18+

```bash
# 1. Clone Citadel once (global install — works across all projects)
git clone https://github.com/SethGammon/Citadel.git ~/citadel

# 2. Launch Claude Code with the plugin loaded
claude --plugin-dir ~/citadel

# 3. Run first-time setup inside any project
/do setup
```

**Persistent install (recommended):**
```
/plugin marketplace add ~/citadel
/plugin install citadel@citadel-local
/reload-plugins
```

After install, `/do` is available in every Claude Code session.

---

## Core Concept: The `/do` Router

Type what you want in plain English. `/do` classifies intent and dispatches to the cheapest capable path automatically.

```
/do fix the typo on line 42          → Direct edit (zero overhead)
/do review the auth module           → /review (5-pass code review)
/do why is the API returning 500     → /systematic-debugging
/do add payments to my app           → /create-app tier 5
/do build me a recipe app            → /prd → /architect → /archon
/do overhaul all three services      → /fleet (parallel agents)
/do continue                         → resumes active campaign
```

You never have to choose the tier. If routing is wrong, say "wrong tool" or "just do it yourself" and it adjusts.

---

## Four-Tier Orchestration Ladder

| Tier | Tool | When to Use |
|------|------|-------------|
| 1 | **Skill** | Single-domain task (review, test, doc, refactor) |
| 2 | **Marshal** | Multi-skill chain within one session |
| 3 | **Archon** | Multi-session autonomous campaign with quality gates |
| 4 | **Fleet** | Parallel agents across isolated git worktrees |

---

## Key Skills Reference

### App Creation

```
/prd "a recipe sharing app with user accounts and ratings"
# → Generates a full Product Requirements Document

/architect
# → Converts active PRD into file tree, build phases, end conditions

/create-app
# → End-to-end creation (5 tiers: blank → guided → templated → generated → feature addition)
```

### Core Skills

```
/review src/auth/           # 5-pass: correctness, security, performance, readability, consistency
/test-gen src/payments.ts   # Generates runnable tests; detects framework; retries up to 3x
/doc-gen src/api/           # Function/module/API reference docs; matches your style
/refactor src/legacy/       # Safe multi-file refactor; typechecks before/after; reverts on failure
/scaffold "UserService"     # Project-aware file generation; reads your conventions
/create-skill               # Turns a repeating pattern into a reusable Citadel skill
```

### Research & Debugging

```
/research "best approach for rate limiting in Express"
/research-fleet "compare auth strategies: JWT vs sessions vs OAuth"  # parallel scouts
/experiment "optimize this function" fitness="benchmark score"        # isolated worktree loops
/systematic-debugging                                                  # 4-phase root cause; stops after 2 failed fixes
```

### Quality & Verification

```
/design                  # Generates/maintains design manifest for visual consistency
/qa                      # Playwright-based browser interaction testing (optional dep)
/postmortem              # Auto-generates structured postmortem from completed campaign
/triage                  # GitHub issue/PR investigator; classifies and reviews contributed code
```

### Utilities

```
/live-preview            # Mid-build visual verification via screenshots
/session-handoff         # Transfers context between sessions
/do setup                # First-run harness configuration
```

---

## Campaign Persistence

Campaigns survive terminal closes and session restarts. State is stored in markdown files under `.planning/` and `.citadel/`.

```bash
# Start a campaign
/do build a full authentication system with JWT and refresh tokens

# Close terminal, come back tomorrow
/do continue             # Picks up exactly where it left off
```

Campaign files track:
- Completed/pending phases
- Decisions made
- Feature status
- Continuation state

See `docs/CAMPAIGNS.md` in the Citadel repo for the full spec.

---

## Fleet: Parallel Agents

Fleet runs 2–3 Claude Code agents simultaneously in isolated git worktrees. Discoveries compress into ~500-token briefs that relay between waves.

```
/fleet "refactor the database layer, API layer, and auth layer in parallel"
```

**How it works internally:**
1. Fleet decomposes work into independent streams
2. Each stream gets an isolated worktree (`git worktree add`)
3. Agents run in parallel; discoveries are extracted and compressed
4. Wave 2 agents receive the compressed brief from wave 1
5. Results merge back to main branch

**Safety:** Parallel agents never share a worktree. Merges are gated on quality checks.

See `docs/FLEET.md` for coordination protocol details.

---

## Hooks (Automatic — No Configuration Needed)

These run without any action on your part:

| Hook | Trigger | Behavior |
|------|---------|----------|
| Per-file typecheck | Every file edit | Catches type errors at write-time |
| Circuit breaker | 3 consecutive tool failures | Forces "try a different approach" |
| Quality gate | Session end | Scans modified files for anti-patterns |
| Intake scanner | Session start | Reports pending work items |
| File protection | Before edit/read | Blocks edits to protected files; blocks `.env` reads |
| Pre-compaction save | Before context compaction | Saves full session state |
| Post-compaction restore | After context compaction | Restores session state |
| Worktree setup | Agent spawn | Auto-installs deps in parallel worktrees |
| External action gate | Before push/PR (opt-in) | Requires user approval before external actions |
| Init project | Session start | Auto-scaffolds `.planning/` and `.citadel/scripts/` |

**Validate all hooks are working:**
```bash
node hooks_src/smoke-test.js
```

---

## Real Usage Patterns

### Pattern 1: Start a New App from Scratch

```
/do build me a todo app with React, TypeScript, and Supabase
```

Citadel will:
1. Run `/prd` to generate requirements
2. Run `/architect` to generate file tree and phases
3. Spawn `/archon` to execute the campaign autonomously
4. Run `/qa` at the end to verify the result

### Pattern 2: Safe Multi-File Refactor

```
/refactor convert all callback-style async code in src/ to async/await
```

Citadel will:
1. Typecheck before changes
2. Apply refactoring across files
3. Typecheck after changes
4. Revert automatically if typechecks fail

### Pattern 3: Test Generation with Retry

```
/test-gen src/services/payments.ts
```

Citadel will:
1. Detect your test framework (Jest, Vitest, Mocha, etc.)
2. Generate tests
3. Run them
4. If failures occur, iterate up to 3 times to fix

### Pattern 4: Parallel Research

```
/research-fleet "compare: Prisma vs Drizzle vs Kysely for a multi-tenant SaaS"
```

Multiple research scouts run in parallel, then compress findings into a unified report.

### Pattern 5: Autonomous Debugging

```
/do why is my Stripe webhook returning 400 intermittently
```

Routes to `/systematic-debugging`, which runs:
1. Symptom classification
2. Hypothesis generation
3. Targeted investigation
4. Fix attempt (with emergency stop after 2 failed fixes)

---

## Creating Custom Skills

Turn any repeating pattern into a Citadel skill:

```
/create-skill "every time I add a new API endpoint I need to: create the route file,
add OpenAPI spec, generate the Zod schema, write integration tests, and update the README"
```

Citadel generates a `.claude/skills/your-skill-name.md` file you can invoke with `/your-skill-name` in future sessions.

---

## Project Structure After Setup

```
your-project/
├── .planning/              # Campaign state (phases, decisions, status)
│   ├── CAMPAIGN.md
│   ├── DECISIONS.md
│   └── FEATURES.md
├── .citadel/
│   └── scripts/            # Per-project hook scripts
└── ...your code...

~/citadel/                  # Plugin install (global)
├── hooks_src/              # Hook implementations
│   └── smoke-test.js
├── skills/                 # All 25 skill definitions
├── agents/                 # Sub-agent definitions
└── docs/
    ├── CAMPAIGNS.md
    ├── FLEET.md
    ├── SKILLS.md
    └── HOOKS.md
```

---

## Token Cost Guide

| Operation | Approximate Cost |
|-----------|-----------------|
| Skills (not loaded) | 0 tokens |
| `/do` router (Tier 3) | ~500 tokens |
| Hook per edit | ~100 tokens |
| The actual work | Varies by task |

---

## Troubleshooting

**`/do` not found after install:**
```
/reload-plugins
```

**Hooks not firing:**
```bash
node ~/citadel/hooks_src/smoke-test.js
```

**Campaign state corrupted:**
```
/do setup          # Re-initializes .planning/ and .citadel/ without destroying code
```

**Fleet agents conflicting:**
Fleet uses isolated `git worktree` per agent — conflicts mean you have uncommitted changes in the main worktree. Commit or stash first:
```bash
git stash
/fleet "..."
```

**Archon stopped mid-campaign:**
```
/do continue       # Reads .planning/CAMPAIGN.md and resumes from last completed phase
```

**Circuit breaker triggered (3 failures):**
This is intentional. Tell Citadel the correct approach explicitly:
```
/do [describe what you actually want, with more specifics]
```

---

## Key Docs

- [QUICKSTART.md](QUICKSTART.md) — Full install guide
- [docs/SKILLS.md](docs/SKILLS.md) — All 25 skills with parameters
- [docs/HOOKS.md](docs/HOOKS.md) — Hook reference and opt-in config
- [docs/CAMPAIGNS.md](docs/CAMPAIGNS.md) — Campaign persistence spec
- [docs/FLEET.md](docs/FLEET.md) — Fleet coordination protocol
- [CONTRIBUTING.md](CONTRIBUTING.md) — How to submit skills and PRs
```
