```markdown
---
name: gbrain-agent-knowledge
description: Skill for using GBrain — a Postgres-backed markdown knowledge brain with hybrid search for AI agents
triggers:
  - set up gbrain for my agent
  - help me build a knowledge brain
  - index my markdown files with gbrain
  - search my notes with an AI agent
  - gbrain import and query
  - set up agent memory with markdown and postgres
  - configure gbrain skillpack
  - connect gbrain to my AI agent
---

# GBrain Agent Knowledge Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

GBrain is a markdown-first, Postgres-backed knowledge brain for AI agents. It turns a git repo of markdown files into a hybrid-searchable (vector + keyword + RRF) knowledge base. The agent reads from and writes to the brain, compounding knowledge over time. Built to work with OpenClaw and Hermes Agent, but usable with any agent that can run CLI commands or call MCP tools.

---

## What GBrain Does

- **Indexes markdown files** into Postgres + pgvector via chunking and OpenAI embeddings
- **Hybrid search** — keyword (tsvector), vector (pgvector), and RRF fusion in one query
- **Knowledge model** — compiled truth header + append-only timeline per page
- **MCP layer** — exposes `gbrain search`, `gbrain get`, `gbrain query` as agent tools
- **Dream cycle** — nightly cron that enriches entities, fixes links, and consolidates memory
- **Works without Postgres** — the schema and skillpack work with plain markdown + grep until scale demands more

---

## Installation

### Prerequisites

| Dependency | Purpose | Source |
|---|---|---|
| Bun | Runtime | `curl -fsSL https://bun.sh/install \| bash` |
| Supabase (Pro) | Postgres + pgvector | [supabase.com](https://supabase.com) |
| OpenAI API key | Embeddings (`text-embedding-3-large`) | `OPENAI_API_KEY` env var |
| Anthropic API key | Multi-query expansion, LLM chunking | `ANTHROPIC_API_KEY` env var |

### Install GBrain

```bash
# Install via bun
bun add github:garrytan/gbrain

# Or globally
bun add -g github:garrytan/gbrain
```

### Environment Variables

```bash
export OPENAI_API_KEY=sk-...
export ANTHROPIC_API_KEY=sk-ant-...
# Supabase URL is configured interactively via: gbrain init --supabase
```

### Initialize

```bash
gbrain init --supabase
# Follow the wizard to connect your Supabase database
# This runs the SQL migrations (pgvector, tsvector, RRF functions)
```

---

## Key CLI Commands

### Import

```bash
# Import a markdown repo (skip embedding for speed, embed in background)
gbrain import ~/git/brain/ --no-embed

# Import and embed immediately
gbrain import ~/Documents/obsidian-vault/

# Import with Obsidian wikilink conversion
gbrain import ~/Documents/obsidian-vault/ --convert-wikilinks
```

### Stats

```bash
gbrain stats
# Pages: 342, Chunks: 1,847, Embedded: 1,847, Links: 0
```

### Search and Query

```bash
# Hybrid search (keyword + vector + RRF)
gbrain query "competitive dynamics in fintech"

# Get a specific page by slug or path
gbrain get people/pedro-franceschi

# Keyword-only search (no embeddings required)
gbrain search "board meeting March"

# Semantic search (requires embeddings)
gbrain query "what have I said about founder resilience?"
```

### Sync

```bash
# Pick up changes from the markdown repo after manual edits
gbrain sync

# Sync a specific directory
gbrain sync ~/git/brain/people/
```

### Discover

```bash
# Scan machine for markdown repos
gbrain discover
# Reports paths, file counts, type (plain markdown vs Obsidian vault)
```

---

## Knowledge Model (Markdown Schema)

Every page follows the **compiled truth + append-only timeline** pattern. Never edit the timeline — always append. Rewrite the compiled truth section when evidence changes.

### Person Page (`people/jordan-smith.md`)

```markdown
# Jordan Smith

**Role:** Partner, Accel
**Company:** [[companies/accel]]
**First met:** 2021-03-15 (YC Demo Day)
**Contact:** jordan@accel.com

## Compiled Truth

Jordan leads Series A in developer tools. Focused on infra and AI-native companies.
Strong network in NYC. Warm intro from Pedro (2022). Key relationship for fundraising.
Prefers Slack over email. Responds fast to concrete asks.

## Open Threads

- [ ] Follow up on the infra fund thesis deck (due 2026-04-15)
- [ ] Intro to their new operating partner

## Timeline

- 2021-03-15 — Met at YC Demo Day. Brief intro. Source: calendar/yc-demo-day-2021
- 2022-06-01 — Pedro intro'd over email. Discussed Series A landscape. Source: email
- 2024-11-20 — Coffee in NYC. Talked about AI-native infra thesis. Source: meeting-transcripts/2024-11-20-jordan
- 2026-04-08 — Zoom: shared deck draft. Positive signal on market size. Source: calendar/zoom-jordan-2026-04-08
```

### Company Page (`companies/accel.md`)

```markdown
# Accel

**Type:** Venture Capital
**Stage:** Series A–C
**HQ:** Palo Alto / London / NYC
**Key contacts:** [[people/jordan-smith]], [[people/rich-wong]]

## Compiled Truth

Top-tier global VC. Strong in developer tools, infra, and fintech.
Accel's NYC presence has grown since 2022. Jordan is the primary contact.
Portfolio includes Brex, Atlassian, Slack.

## Timeline

- 2022-06-01 — First contact via Jordan intro. Source: email
- 2024-11-20 — Jordan meeting. Discussed fund cycle and portfolio fit. Source: meeting-transcripts/2024-11-20-jordan
```

### Original Idea Page (`originals/shame-founder-performance.md`)

```markdown
# Shame and Founder Performance

**Thesis category:** Psychology / Leadership
**Created:** 2025-08-12
**Status:** Developing

## Compiled Truth

Founders who operate from shame (fear of judgment, proving worth) hit a ceiling earlier
than founders who operate from curiosity. Shame optimizes for external validation;
curiosity optimizes for truth. The transition from shame-driven to curiosity-driven is
often the unlock at the Series B inflection point.

## Evidence

- 2025-08-12 — Observed in 3 portfolio companies simultaneously. Source: notes/coaching-session-aug-2025
- 2025-09-04 — Brené Brown's research on shame vs guilt maps onto this. Source: media/brene-brown-daring-greatly
- 2026-01-17 — Jordan independently raised something similar in our meeting. Source: meeting-transcripts/2026-01-17-jordan
```

---

## Agent Integration Patterns

### The Brain-Agent Loop

```
Signal arrives (meeting, email, tweet, note)
  → Agent detects entities (people, companies, ideas)
  → READ: gbrain search / gbrain get (check brain first)
  → Respond with full context
  → WRITE: update markdown pages with new information
  → gbrain sync (index changes)
```

### Entity Detection (spawn on every message)

When a message mentions a person, company, or original idea:

1. `gbrain get people/<slug>` — check if page exists
2. If exists: read compiled truth, check open threads
3. If not exists: create new page from schema template
4. After conversation: append new facts to timeline, rewrite compiled truth if needed
5. `gbrain sync` to re-index

### Meeting Ingestion (7-step enrichment)

```
1. gbrain search <person> — pull dossier before meeting
2. During/after: capture transcript → meeting-transcripts/<date>-<person>.md
3. Extract entities mentioned: people, companies, ideas
4. For each entity: gbrain get <entity> → append timeline entry
5. Update compiled truth for each entity page
6. Create new pages for unknown entities
7. gbrain sync
```

### MCP Tool Usage (OpenClaw / Hermes)

When installed as an MCP skill, these tools are available to the agent:

```
gbrain_search(query: string) → ranked list of pages
gbrain_get(slug: string) → full page content
gbrain_query(query: string) → hybrid search with scores
gbrain_sync() → re-index changed files
```

---

## Programmatic Usage (TypeScript)

```typescript
import { GBrain } from 'gbrain'

const brain = new GBrain({
  supabaseUrl: process.env.SUPABASE_URL!,
  supabaseKey: process.env.SUPABASE_SERVICE_KEY!,
  openaiApiKey: process.env.OPENAI_API_KEY!,
  anthropicApiKey: process.env.ANTHROPIC_API_KEY,
})

// Hybrid search
const results = await brain.query('competitive dynamics fintech', { limit: 5 })
for (const r of results) {
  console.log(r.slug, r.score, r.excerpt)
}

// Get a specific page
const page = await brain.get('people/jordan-smith')
console.log(page.content)

// Import a directory
await brain.import('/Users/me/git/brain', { embed: true })

// Sync changes
await brain.sync()

// Get stats
const stats = await brain.stats()
console.log(`Pages: ${stats.pages}, Chunks: ${stats.chunks}, Embedded: ${stats.embedded}`)
```

---

## The Dream Cycle (Nightly Cron)

The dream cycle runs while you sleep and enriches the brain autonomously.

### What it does
1. Scans all conversations from the day
2. Detects unresolved entity references (people/companies without pages)
3. Enriches missing entities (creates stubs, pulls public data)
4. Fixes broken `[[wikilinks]]`
5. Consolidates duplicate timeline entries
6. Re-embeds pages that changed

### Cron setup (`crontab -e`)

```cron
# Run dream cycle nightly at 2am
0 2 * * * /Users/me/.bun/bin/gbrain dream >> /tmp/gbrain-dream.log 2>&1
```

### Dream cycle command

```bash
gbrain dream
# Scans today's activity, enriches entities, syncs
gbrain dream --dry-run
# Shows what would change without writing
```

---

## Directory Layout (Recommended)

```
brain/
├── people/           # One page per person
├── companies/        # One page per company/org
├── meetings/         # Meeting notes (separate from transcripts)
├── meeting-transcripts/  # Raw transcripts with AI analysis
├── originals/        # Your original ideas, organized by thesis
├── calendar/         # Calendar event stubs with entity links
├── notes/            # Apple Notes or freeform notes
├── media/            # Video transcripts, book summaries, articles
├── food/             # Food and restaurant guide (optional)
├── travel/           # Travel logs (optional)
└── DREAMS.md         # Dream cycle log (auto-maintained)
```

---

## Hybrid Search: How It Works

GBrain fuses three retrieval signals using Reciprocal Rank Fusion (RRF):

| Signal | How | Best for |
|---|---|---|
| **Vector** | pgvector cosine similarity on `text-embedding-3-large` chunks | Semantic meaning, concepts |
| **Keyword** | Postgres `tsvector` full-text search | Exact names, dates, terms |
| **RRF** | Rank fusion of both signals | Combined — used by default |

```sql
-- What GBrain runs under the hood (simplified)
WITH vector_results AS (
  SELECT slug, ROW_NUMBER() OVER (ORDER BY embedding <=> $query_embedding) AS rank
  FROM chunks
),
keyword_results AS (
  SELECT slug, ROW_NUMBER() OVER (ORDER BY ts_rank(tsv, $query_tsquery) DESC) AS rank
  FROM chunks
  WHERE tsv @@ $query_tsquery
),
rrf AS (
  SELECT slug,
    COALESCE(1.0/(60+v.rank),0) + COALESCE(1.0/(60+k.rank),0) AS score
  FROM vector_results v
  FULL OUTER JOIN keyword_results k USING (slug)
)
SELECT slug, score FROM rrf ORDER BY score DESC LIMIT 10;
```

---

## Layered Memory Model

| Layer | What it stores | How to query |
|---|---|---|
| **gbrain** | People, companies, meetings, ideas, media | `gbrain search`, `gbrain query`, `gbrain get` |
| **Agent memory** (`memory_search`) | Preferences, decisions, operational config | OpenClaw / Hermes memory tools |
| **Session context** | Current conversation | Automatic (in-context) |

Always check all three. GBrain for world facts. Agent memory for operational config. Session for immediate context.

---

## Troubleshooting

### Embeddings not running

```bash
# Check OPENAI_API_KEY is set
echo $OPENAI_API_KEY

# Force re-embed all chunks
gbrain embed --all

# Check embedding status
gbrain stats
```

### Search returns no results

```bash
# Check if pages were imported
gbrain stats

# Run keyword-only search to confirm data is there
gbrain search "exact term you expect"

# If chunks exist but embeddings are 0, run:
gbrain embed
```

### Sync not picking up changes

```bash
# Force full re-sync
gbrain sync --full

# Sync a specific file
gbrain sync ~/git/brain/people/jordan-smith.md
```

### pgvector extension missing

```sql
-- Run in Supabase SQL editor
create extension if not exists vector;
```

### Wikilinks not resolving

```bash
# Re-run link resolution pass
gbrain sync --relink

# Check for broken links
gbrain links --broken
```

### Large repo is slow to import

```bash
# Import without embedding first (fast), embed in background
gbrain import ~/git/brain/ --no-embed

# Then embed incrementally
gbrain embed --batch-size 50
```

---

## OpenClaw / Hermes Agent Install

Paste this into your agent to trigger automated setup:

```
Set up gbrain (https://github.com/garrytan/gbrain) as my knowledge brain.

1. Install bun if needed: curl -fsSL https://bun.sh/install | bash
   Then: bun add github:garrytan/gbrain

2. Run: gbrain init --supabase

3. Discover markdown repos: gbrain discover
   Import the best candidate: gbrain import <path> --no-embed

4. Run: gbrain query "<something relevant to my content>"

5. Read docs/GBRAIN_RECOMMENDED_SCHEMA.md and restructure if needed

6. Read docs/GBRAIN_SKILLPACK.md and update your skills accordingly
```

Or via skill install (OpenClaw):

```bash
openclaw skills install gbrain
```
```
