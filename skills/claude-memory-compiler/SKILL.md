```markdown
---
name: claude-memory-compiler
description: Give Claude Code an evolving memory that captures sessions, extracts decisions and lessons via the Claude Agent SDK, and compiles everything into structured cross-referenced knowledge articles.
triggers:
  - set up claude memory compiler
  - give claude code persistent memory
  - capture my coding sessions automatically
  - compile my claude conversations into a knowledge base
  - set up session hooks for claude code
  - install llm knowledge base for my project
  - make claude remember decisions across sessions
  - auto-capture claude code session transcripts
---

# Claude Memory Compiler

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Claude Memory Compiler gives Claude Code a persistent, evolving memory. Claude Code hooks automatically capture conversation transcripts at session end or pre-compaction. A background process using the Claude Agent SDK extracts decisions, lessons, and patterns, appending them to daily logs. A compiler then organizes those logs into structured, cross-referenced knowledge articles — no vector database or RAG required at personal scale.

Inspired by [Karpathy's LLM Knowledge Base](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) architecture.

---

## Installation

### Option 1 — Tell your AI agent

```
Clone https://github.com/coleam00/claude-memory-compiler into this project.
Set up the Claude Code hooks so my conversations automatically get captured
into daily logs, compiled into a knowledge base, and injected back into
future sessions. Read the AGENTS.md for the full technical reference.
```

### Option 2 — Manual setup

```bash
# Clone into your project root
git clone https://github.com/coleam00/claude-memory-compiler .claude-memory

cd .claude-memory

# Install dependencies (requires uv)
uv sync

# Copy hooks config into your project's Claude settings
cp .claude/settings.json ../.claude/settings.json
# Or merge the hooks block into an existing .claude/settings.json
```

### Dependency requirements

- Python 3.10+
- [`uv`](https://github.com/astral-sh/uv) package manager
- Claude Code (with Max, Team, or Enterprise subscription — no separate API credits needed)

---

## How It Works

```
Conversation
  -> SessionEnd / PreCompact hooks
  -> flush.py  (Claude Agent SDK extracts knowledge)
  -> daily/YYYY-MM-DD.md
  -> compile.py (after 6 PM or manually)
  -> knowledge/concepts/, connections/, qa/
  -> SessionStart hook injects index.md into next session
  -> cycle repeats
```

### Hook events used

| Hook | Purpose |
|------|---------|
| `SessionEnd` | Primary capture — fires when a session closes |
| `PreCompact` | Safety net — fires before mid-session compaction |
| `SessionStart` | Injects the knowledge index into each new session |

---

## Key Commands

```bash
# Compile new daily logs into knowledge articles
uv run python scripts/compile.py

# Ask a question against the knowledge base
uv run python scripts/query.py "Why did we switch from SQLite to Postgres?"

# Ask + save the answer back into the knowledge base
uv run python scripts/query.py "What auth strategy are we using?" --file-back

# Run all 7 health checks (broken links, orphans, contradictions, staleness)
uv run python scripts/lint.py

# Run only free structural checks (no LLM calls)
uv run python scripts/lint.py --structural-only
```

---

## Directory Structure

```
your-project/
├── .claude/
│   └── settings.json          # Hook configuration
├── daily/
│   └── YYYY-MM-DD.md          # Raw daily session logs
├── knowledge/
│   ├── index.md               # Master index (injected at session start)
│   ├── concepts/              # Structured concept articles
│   ├── connections/           # Cross-reference articles
│   └── qa/                    # Q&A articles saved with --file-back
└── scripts/
    ├── flush.py               # Session capture + extraction
    ├── compile.py             # Daily logs -> knowledge articles
    ├── query.py               # Index-guided retrieval
    └── lint.py                # Knowledge base health checks
```

---

## Configuration — `.claude/settings.json`

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "command": "uv run python scripts/flush.py",
        "cwd": "${workspaceFolder}"
      }
    ],
    "PreCompact": [
      {
        "command": "uv run python scripts/flush.py --pre-compact",
        "cwd": "${workspaceFolder}"
      }
    ],
    "SessionStart": [
      {
        "command": "uv run python scripts/inject.py",
        "cwd": "${workspaceFolder}"
      }
    ]
  }
}
```

If you already have a `.claude/settings.json`, merge only the `hooks` block — don't overwrite existing settings.

---

## Code Examples

### Manually flush a transcript

```python
# scripts/flush.py is called automatically by hooks, but you can invoke it directly
import subprocess

result = subprocess.run(
    ["uv", "run", "python", "scripts/flush.py"],
    capture_output=True,
    text=True
)
print(result.stdout)
```

### Programmatic query

```python
# Query the knowledge base from your own scripts
import subprocess
import sys

def query_knowledge_base(question: str, save_back: bool = False) -> str:
    cmd = ["uv", "run", "python", "scripts/query.py", question]
    if save_back:
        cmd.append("--file-back")
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.stdout

answer = query_knowledge_base("What database migration strategy are we using?")
print(answer)
```

### Trigger compilation manually

```python
import subprocess

def compile_knowledge():
    """Compile all unprocessed daily logs into knowledge articles."""
    result = subprocess.run(
        ["uv", "run", "python", "scripts/compile.py"],
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        print("Compile error:", result.stderr)
    else:
        print(result.stdout)

compile_knowledge()
```

### Run lint checks programmatically

```python
import subprocess

def lint_knowledge_base(structural_only: bool = False) -> dict:
    cmd = ["uv", "run", "python", "scripts/lint.py"]
    if structural_only:
        cmd.append("--structural-only")
    result = subprocess.run(cmd, capture_output=True, text=True)
    return {
        "passed": result.returncode == 0,
        "output": result.stdout,
        "errors": result.stderr
    }

report = lint_knowledge_base(structural_only=True)
print(report["output"])
```

---

## Daily Log Format (`daily/YYYY-MM-DD.md`)

Each flush appends an entry like:

```markdown
## Session — 2026-04-10 14:32

### Decisions
- Switched auth from JWT to session cookies to simplify SSR compatibility.

### Lessons Learned
- `prisma migrate dev` resets the shadow DB on every run — use `--skip-generate` in CI.

### Patterns Identified
- All new API routes follow the `app/api/[resource]/route.ts` convention.

### Gotchas
- The `useRouter` hook from `next/navigation` behaves differently in Server Components — always import from the correct package.
```

---

## Knowledge Article Format (`knowledge/concepts/`)

Compiled articles use structured markdown:

```markdown
# Authentication Strategy

**Last Updated:** 2026-04-10
**Related:** [[Session Management]], [[API Security]]

## Summary
We use session cookies (not JWTs) for authentication to simplify SSR compatibility with Next.js App Router.

## Key Decisions
- Chose session cookies over JWTs on 2026-04-08 after discovering JWT refresh complexity with RSC.

## Lessons
- Always import `useRouter` from `next/navigation` in client components, never from `next/router`.

## Open Questions
- Should we add refresh token rotation for long-lived sessions?
```

---

## Automatic Compilation Trigger

`flush.py` checks local time after each session. If it is **after 6 PM**, it automatically triggers `compile.py` to process that day's logs. This means no manual scheduling is needed for end-of-day compilation.

To compile at any time:

```bash
uv run python scripts/compile.py
```

---

## Why No RAG / Vector Database?

At personal scale (50–500 articles), the LLM reads `knowledge/index.md` directly and understands context better than cosine similarity over embeddings. RAG becomes necessary only at ~2,000+ articles when the index exceeds the context window.

The index file (`knowledge/index.md`) is a plain markdown table of contents injected at session start — giving Claude full awareness of what's been learned without any retrieval infrastructure.

---

## Subscription & Cost Notes

- Runs on **Claude Max, Team, or Enterprise** subscriptions — no separate API billing required
- Anthropic has confirmed personal use of the Claude Agent SDK is covered under existing subscriptions
- `lint.py --structural-only` runs broken-link and orphan checks **without any LLM calls** (free)
- LLM-powered lint checks (contradiction detection, staleness) consume subscription usage

---

## Common Patterns

### Pattern: Bootstrap a new project

```bash
# 1. Clone the compiler into your project
git clone https://github.com/coleam00/claude-memory-compiler .

# 2. Install dependencies
uv sync

# 3. Merge hooks into .claude/settings.json (see Configuration section)

# 4. Open Claude Code — hooks activate automatically on next session
```

### Pattern: Recover after a missed session

If hooks failed to fire, manually flush by running `flush.py` with the transcript path (check AGENTS.md for the exact CLI flags your version supports):

```bash
uv run python scripts/flush.py --transcript path/to/transcript.json
```

### Pattern: Keep the knowledge base healthy

Run this weekly:

```bash
# Free structural check first
uv run python scripts/lint.py --structural-only

# Full check (uses LLM credits) when issues are suspected
uv run python scripts/lint.py
```

### Pattern: Answer a question and persist it

```bash
uv run python scripts/query.py "What's our strategy for handling database migrations in CI?" --file-back
# Answer is printed AND saved to knowledge/qa/YYYY-MM-DD-slug.md
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Hooks not firing | Confirm `.claude/settings.json` exists in your project root with the correct `hooks` block |
| `uv` not found | Install uv: `curl -Lsf https://astral.sh/uv/install.sh \| sh` |
| `uv sync` fails | Ensure Python 3.10+ is active; run `uv python install 3.12` if needed |
| Compilation produces empty articles | Check `daily/` for logs; flush may not have run — open and close a Claude Code session |
| `query.py` returns vague answers | Run `compile.py` first to ensure daily logs are compiled into the index |
| Lint reports broken links | Article was renamed or deleted; update cross-references in the affected files |
| Session injection not working | Verify the `SessionStart` hook is present in settings and `knowledge/index.md` exists |

---

## Full Technical Reference

See **[AGENTS.md](https://github.com/coleam00/claude-memory-compiler/blob/main/AGENTS.md)** in the repository for:

- Complete article schema definitions
- Hook architecture internals
- Script CLI flags and extension points
- Cross-platform (Windows/macOS/Linux) hook path details
- Customization options for extraction prompts
- Cost breakdown per operation
```
