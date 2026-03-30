```markdown
---
name: openclaw-auto-dream
description: Automatic cognitive memory consolidation for OpenClaw/MyClaw agents — sleep cycles, importance scoring, forgetting curves, knowledge graphs, and health dashboards.
triggers:
  - set up auto-dream memory for my openclaw agent
  - configure memory consolidation for myclaw
  - how do I install openclaw auto-dream
  - my ai agent keeps forgetting things between sessions
  - set up dream cycles for memory management
  - how does auto-dream importance scoring work
  - export or import memory bundle between myclaw instances
  - show me the memory health dashboard
---

# OpenClaw Auto-Dream

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

OpenClaw Auto-Dream is a cognitive memory architecture for [OpenClaw](https://github.com/openclaw/openclaw) agents (as used on [MyClaw.ai](https://myclaw.ai)). It gives your AI agent the ability to periodically "sleep and dream" — scanning raw daily logs, extracting structured knowledge, scoring importance, applying forgetting curves, building a knowledge graph, and surfacing non-obvious insights. The result is an agent that genuinely learns and connects the dots over time rather than accumulating stale, disconnected files.

---

## How It Works

Auto-Dream runs a **three-phase dream cycle** (default: 4 AM daily via cron):

1. **Collect** — Scans unconsolidated daily logs (last 7 days), detects priority markers, extracts decisions/people/facts/projects/lessons/procedures/open threads.
2. **Consolidate** — Routes each insight to one of five memory layers, deduplicates semantically, assigns unique IDs (`mem_NNN`), creates relation links.
3. **Evaluate** — Scores importance, applies forgetting curves, calculates a 5-metric health score, generates insights, writes dream report, sends notification.

### Five Memory Layers

| Layer | Storage | Purpose |
|-------|---------|---------|
| Working | LCM plugin (auto-detected) | Real-time context compression & semantic recall |
| Episodic | `memory/episodes/*.md` | Project narratives, event timelines |
| Long-term | `MEMORY.md` | Facts, decisions, people, milestones, strategy |
| Procedural | `memory/procedures.md` | Workflows, preferences, tool patterns |
| Index | `memory/index.json` | Metadata, scores, relations, health stats |

---

## Installation

### Via ClawHub (Recommended)

Tell your MyClaw agent:

```
Install the openclaw-auto-dream skill from ClawHub
```

Or manually inside your OpenClaw agent environment:

```bash
claw skill install openclaw-auto-dream
```

### Manual Installation

Clone into your OpenClaw skills directory:

```bash
git clone https://github.com/LeoYeAI/openclaw-auto-dream.git \
  ~/.openclaw/skills/openclaw-auto-dream
```

Then register the skill with your agent:

```bash
claw skill register ~/.openclaw/skills/openclaw-auto-dream
```

### First-Time Setup

After installation, tell your agent:

```
Set up auto-dream
```

The setup wizard will:
- Detect whether the optional LCM plugin is available (for the Working memory layer)
- Create the `memory/` directory structure
- Initialize `memory/index.json` with default health metrics
- Ask for your preferred notification level (`silent` / `summary` / `full`)
- Schedule the dream cron job (default: `0 4 * * *`)

---

## Configuration

Auto-Dream is configured via `memory/config.json` in your agent's workspace:

```json
{
  "dream_schedule": "0 4 * * *",
  "notification_level": "summary",
  "scan_window_days": 7,
  "forgetting": {
    "min_age_days": 90,
    "importance_threshold": 0.3
  },
  "scoring": {
    "recency_half_life_days": 180,
    "permanent_marker": "⚠️ PERMANENT",
    "high_marker": "🔥 HIGH",
    "pin_marker": "📌 PIN"
  },
  "layers": {
    "working_lcm": true,
    "episodic": true,
    "long_term": true,
    "procedural": true
  },
  "export": {
    "compress": true,
    "include_archive": false
  }
}
```

### Environment Variables

If you need to override config values via the environment (e.g. in CI or multi-instance deployments):

```bash
AUTODREAM_SCHEDULE="0 2 * * *"          # Override cron schedule
AUTODREAM_NOTIFY_LEVEL="full"           # silent | summary | full
AUTODREAM_SCAN_DAYS=14                  # Days of logs to scan per cycle
AUTODREAM_FORGET_THRESHOLD=0.25         # Importance below which entries are archived
```

---

## Key Commands (Agent Natural Language)

These phrases trigger built-in Auto-Dream intents inside your OpenClaw agent:

| Phrase | Action |
|--------|--------|
| `"Dream now"` | Trigger an immediate full dream cycle |
| `"Show memory dashboard"` | Generate and open the interactive HTML dashboard |
| `"What do you remember about [topic]?"` | Semantic search across all memory layers |
| `"Memory health"` | Print current 5-metric health score |
| `"Export memory bundle"` | Export all layers to `memory/export-YYYY-MM-DD.json` |
| `"Import memory bundle"` | Merge an exported bundle into current memory |
| `"Export only procedures"` | Selective single-layer export |
| `"Forget [topic]"` | Immediately archive entries matching topic |
| `"Pin this"` | Mark current context with `📌 PIN` (immune to forgetting) |
| `"What did you learn last week?"` | Show insights from the last 7 dream logs |

---

## Priority Markers in Daily Logs

Auto-Dream scans your agent's daily log files for special markers during the Collect phase. Use these in any log entry or conversation note:

```markdown
<!-- important -->
Decided to use Postgres over SQLite for the user DB — scalability concern.

⚠️ PERMANENT
Client prefers all reports in US Letter format, not A4.

🔥 HIGH
The deploy pipeline breaks when NODE_ENV is not explicitly set.

📌 PIN
Weekly sync with Alex every Tuesday at 10 AM.
```

| Marker | Effect |
|--------|--------|
| `<!-- important -->` | Extracted and routed to appropriate memory layer |
| `⚠️ PERMANENT` | Always scores `1.0` importance; never archived |
| `🔥 HIGH` | Base weight doubled during importance scoring |
| `📌 PIN` | Immune to forgetting curve; always retained |

---

## Importance Scoring

Every memory entry is scored on each dream cycle:

```
importance = (base_weight × recency_factor × reference_boost) / 8.0
```

Where:
- `recency_factor = max(0.1, 1.0 - days_since_created / 180)`
- `reference_boost = log₂(reference_count + 1)`
- `base_weight` doubles for `🔥 HIGH` entries; `⚠️ PERMANENT` always returns `1.0`

### Example (Python-style pseudocode)

```python
import math

def score_entry(entry: dict, today_date) -> float:
    if "⚠️ PERMANENT" in entry.get("markers", []):
        return 1.0

    days_old = (today_date - entry["created_at"]).days
    recency = max(0.1, 1.0 - days_old / 180)

    refs = entry.get("reference_count", 0)
    ref_boost = math.log2(refs + 1) if refs > 0 else 1.0

    base = entry.get("base_weight", 1.0)
    if "🔥 HIGH" in entry.get("markers", []):
        base *= 2.0

    return min(1.0, (base * recency * ref_boost) / 8.0)
```

---

## Forgetting Curve & Archival

Entries are **never deleted** — only gracefully archived:

```python
def should_archive(entry: dict, today_date) -> bool:
    # Immune markers
    immune = {"⚠️ PERMANENT", "📌 PIN"}
    if immune & set(entry.get("markers", [])):
        return False

    days_unreferenced = (today_date - entry["last_referenced"]).days
    importance = entry["importance_score"]

    return days_unreferenced > 90 and importance < 0.3

def archive_entry(entry: dict):
    summary = f"[{entry['id']}] {entry['title']} — {entry['one_line_summary']}"
    append_to_file("memory/archive.md", summary)
    entry["status"] = "archived"
    # Original ID preserved for relation tracking
```

---

## Health Score

```
health = (
    freshness   × 0.25 +
    coverage    × 0.25 +
    coherence   × 0.20 +
    efficiency  × 0.15 +
    reachability× 0.15
) × 100
```

| Metric | Definition |
|--------|-----------|
| **Freshness** | % of entries referenced in the last 30 days |
| **Coverage** | % of knowledge categories updated in the last 14 days |
| **Coherence** | % of entries with at least one relation link |
| **Efficiency** | Inversely proportional to `MEMORY.md` line count (penalises bloat) |
| **Reachability** | Knowledge graph connectivity via union-find across entry relations |

Check health anytime:

```
"Memory health"
→ 🩺 Health: 79/100
   Freshness: 0.81  Coverage: 0.74  Coherence: 0.68
   Efficiency: 0.91  Reachability: 0.72
   ⚠️ Coherence declining — consider linking isolated entries
```

---

## Cross-Instance Memory Migration

Export from one MyClaw instance:

```
"Export memory bundle"
→ Saved: memory/export-2026-03-28.json (142 entries, 3 layers)
```

The export format:

```json
{
  "version": "4.0",
  "exported_at": "2026-03-28T06:00:00Z",
  "source_instance": "myclaw-instance-abc123",
  "layers": {
    "long_term": [ /* entries */ ],
    "procedural": [ /* entries */ ],
    "episodic": [ /* entries */ ]
  },
  "index": { /* metadata + scores */ }
}
```

Import on another instance (newer entry wins on conflict):

```
"Import memory bundle"
→ Upload or paste path to export JSON
→ Merging 142 entries... 138 new, 4 conflicts resolved (newer wins)
→ Pre-import backup saved to memory/pre-import-backup-2026-03-28.json
```

Selective export:

```
"Export only procedures"
→ Saved: memory/export-procedures-2026-03-28.json (23 entries)
```

---

## Memory File Structure

After setup, your agent workspace will contain:

```
workspace/
├── MEMORY.md                        # Long-term memory layer (human-readable)
├── memory/
│   ├── config.json                  # Auto-Dream configuration
│   ├── index.json                   # Entry metadata, scores, relations, health
│   ├── procedures.md                # Procedural memory layer
│   ├── archive.md                   # Archived (forgotten) entries — one-liners
│   ├── dream-log.md                 # Appended report after each cycle
│   ├── dashboard.html               # Generated interactive dashboard
│   ├── episodes/
│   │   ├── project-alpha.md
│   │   └── onboarding-2026-01.md
│   └── exports/
│       └── export-2026-03-28.json
└── logs/
    ├── daily-2026-03-28.md          # Raw daily logs (scanned by Auto-Dream)
    └── daily-2026-03-27.md
```

---

## Interactive Dashboard

Generate the zero-dependency HTML dashboard:

```
"Show memory dashboard"
→ Generated: memory/dashboard.html
```

The dashboard includes:
- **Animated health gauge** (0–100)
- **5 metric cards** with colour-coded status (green ≥ 0.8, amber ≥ 0.6, red < 0.6)
- **Memory distribution donut chart** (entries per layer)
- **Importance histogram** (score distribution across all entries)
- **Health trend line chart** (last 30 dream cycles)
- **Force-directed knowledge graph** (linked entries visualised)
- **Recent changes**, **dream insights**, **stale entry warnings**

Open it in any browser — no server required, all data embedded inline.

---

## Dream Log Format

After each cycle, a report is appended to `memory/dream-log.md`:

```markdown
## Dream Cycle — 2026-03-28T04:00:12Z

**Health:** 82/100 (↑3 from last cycle)
**Entries:** +5 new · ~3 updated · -1 archived
**Duration:** 14.2s

### New Memories
- mem_087: Decided to migrate auth to Clerk (episodic + long-term)
- mem_088: Alex prefers async updates over meetings (long-term)

### Archived
- mem_034: [compressed] Early Stripe test key config — 2025-09-01

### Insights
1. Project Beta's scope decisions mirror what caused delays in Project Alpha — pattern detected.
2. Strategic decisions cluster on Mondays — likely tied to weekly planning sessions.
3. No lessons recorded for last 3 completed projects — retrospective habit may be slipping.

### Suggestions
- Link mem_072 (auth strategy) ↔ mem_087 (Clerk migration) — high semantic overlap
- mem_091 unreferenced for 78 days — consider pinning or letting it expire
```

---

## Notification Levels

Configure in `memory/config.json` or tell your agent `"Set notifications to [level]"`:

| Level | What You Receive |
|-------|-----------------|
| `silent` | Nothing pushed — logged to `dream-log.md` only |
| `summary` | `🌀 Health: 82/100 \| +5 new, ~3 updated, -1 archived \| 💡 Top insight` |
| `full` | Complete dream report with all sections pushed to your chat channel |

Push is delivered to whichever channel your MyClaw agent is connected to (Telegram, Discord, Slack, WhatsApp, etc.).

---

## Troubleshooting

### Dream cycle not running

```bash
# Check cron is registered
crontab -l | grep autodream

# Run a manual cycle and watch output
claw skill run openclaw-auto-dream --trigger dream --verbose

# Check for log errors
tail -50 ~/.openclaw/logs/skills.log | grep auto-dream
```

### `memory/index.json` is corrupted or missing

```
"Rebuild memory index"
→ Scans all memory layer files and regenerates index.json from scratch
→ Note: reference counts and relation links will be reset
```

### Health score stuck low despite active use

Common causes and fixes:

| Symptom | Cause | Fix |
|---------|-------|-----|
| Coherence < 0.5 | Many unlinked entries | `"Link related memories"` — agent will suggest and apply links |
| Freshness < 0.5 | Old entries not being referenced | `"What do you remember about X?"` — queries boost reference counts |
| Efficiency < 0.5 | `MEMORY.md` has grown too large | `"Compress long-term memory"` — consolidates verbose entries |
| Reachability < 0.5 | Isolated knowledge clusters | Check dream insights — they will flag disconnected subgraphs |

### LCM plugin not detected (Working layer disabled)

```
"Check working memory status"
→ ⚠️ LCM plugin not found — Working layer disabled
→ Install with: claw plugin install lcm
→ Then: claw skill reconfigure openclaw-auto-dream
```

### Entries marked PERMANENT are being archived

This is a bug — `⚠️ PERMANENT` entries should never be archived. Force a fix:

```bash
# In your agent workspace
claw skill run openclaw-auto-dream --repair-permanent-markers
```

---

## Common Patterns

### Mark important decisions automatically in daily logs

Add to your agent's daily log template in `memory/procedures.md`:

```markdown
## Decision Template
When recording a decision, always include:
🔥 HIGH
Decision: [what was decided]
Rationale: [why]
Alternatives considered: [what else was evaluated]
```

### Trigger a dream before a big project review

```
"Dream now, then show me everything you remember about Project Alpha"
```

### Weekly memory hygiene check

```
"Memory health"
"What's stale?"
"Link related memories"
"Dream now"
```

### Multi-instance workflow (e.g. work vs. personal MyClaw)

```
# On work instance
"Export only procedures"
# Copy memory/export-procedures-2026-03-28.json to personal instance
# On personal instance
"Import memory bundle"
# Select the procedures export — workflows sync across instances
```
```
