```markdown
---
name: claude-usage-dashboard
description: Local dashboard for tracking Claude Code token usage, costs, and session history from JSONL logs
triggers:
  - track claude code usage
  - show claude token usage
  - claude code cost dashboard
  - how much am I spending on claude
  - visualize claude sessions
  - claude usage statistics
  - monitor claude code tokens
  - set up claude usage tracking
---

# Claude Code Usage Dashboard

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A zero-dependency local dashboard that reads Claude Code's JSONL session logs and turns them into charts, cost estimates, and usage summaries. Works on API, Pro, and Max plans.

---

## What it does

Claude Code writes detailed usage logs to `~/.claude/projects/` regardless of subscription type. This tool:

- **Parses** those JSONL files into a local SQLite database at `~/.claude/usage.db`
- **Estimates costs** using Anthropic API pricing (April 2026)
- **Serves a browser dashboard** at `http://localhost:8080` with Chart.js charts
- **Tracks** input tokens, output tokens, cache creation tokens, cache read tokens, model used, and session/project metadata

Captures usage from Claude Code CLI, VS Code extension, and Dispatched Code sessions. Does **not** capture Cowork sessions (server-side, no local transcripts).

---

## Installation

No pip, no venv, no build step. Requires Python 3.8+ (standard library only).

```bash
git clone https://github.com/phuryn/claude-usage
cd claude-usage
```

---

## Key CLI Commands

```bash
# macOS/Linux
python3 cli.py dashboard    # scan + open browser dashboard at http://localhost:8080
python3 cli.py scan         # parse JSONL files, populate ~/.claude/usage.db
python3 cli.py today        # print today's usage summary by model
python3 cli.py stats        # print all-time statistics

# Windows
python cli.py dashboard
python cli.py scan
python cli.py today
python cli.py stats
```

The scanner is **incremental** — it tracks each file's path and modification time, so re-running `scan` is fast (only processes new or changed files).

---

## File Structure

| File | Purpose |
|------|---------|
| `scanner.py` | Parses `~/.claude/projects/**/*.jsonl`, writes to SQLite |
| `dashboard.py` | Serves single-page HTML/JS dashboard on `localhost:8080` |
| `cli.py` | Entry point for `scan`, `today`, `stats`, `dashboard` commands |

---

## How Claude Code JSONL Logs Work

Each session creates one JSONL file in `~/.claude/projects/`. Each line is a JSON record. The scanner looks for `assistant`-type records:

```json
{
  "type": "assistant",
  "message": {
    "model": "claude-sonnet-4-6",
    "usage": {
      "input_tokens": 1234,
      "output_tokens": 567,
      "cache_creation_input_tokens": 890,
      "cache_read_input_tokens": 4321
    }
  }
}
```

---

## Cost Pricing Table (April 2026 API Prices)

| Model | Input | Output | Cache Write | Cache Read |
|-------|-------|--------|------------|-----------|
| claude-opus-4-6 | $6.15/MTok | $30.75/MTok | $7.69/MTok | $0.61/MTok |
| claude-sonnet-4-6 | $3.69/MTok | $18.45/MTok | $4.61/MTok | $0.37/MTok |
| claude-haiku-4-5 | $1.23/MTok | $6.15/MTok | $1.54/MTok | $0.12/MTok |

Only models whose name contains `opus`, `sonnet`, or `haiku` are costed. Others show `n/a`.

> **Note:** These are API prices. Max/Pro subscribers pay subscription rates, not per-token.

---

## Working with the Scanner Programmatically

```python
from scanner import Scanner

# Scan all Claude Code JSONL logs into ~/.claude/usage.db
scanner = Scanner()
scanner.scan()

# Access the SQLite database directly
import sqlite3
from pathlib import Path

db_path = Path.home() / ".claude" / "usage.db"
conn = sqlite3.connect(db_path)
cursor = conn.cursor()

# Get total tokens by model
cursor.execute("""
    SELECT model,
           SUM(input_tokens) as total_input,
           SUM(output_tokens) as total_output,
           SUM(cache_read_input_tokens) as total_cache_read
    FROM usage
    GROUP BY model
    ORDER BY total_input DESC
""")
rows = cursor.fetchall()
for row in rows:
    print(row)

conn.close()
```

---

## Working with the Dashboard Server Programmatically

```python
from dashboard import DashboardServer

# Start the dashboard on a custom port
server = DashboardServer(port=9090)
server.serve()
# Opens http://localhost:9090 in browser
```

---

## Querying Usage Data Directly (SQLite)

```python
import sqlite3
from pathlib import Path
from datetime import date, timedelta

db_path = Path.home() / ".claude" / "usage.db"
conn = sqlite3.connect(db_path)
cursor = conn.cursor()

# Today's usage
today = date.today().isoformat()
cursor.execute("""
    SELECT model,
           SUM(input_tokens),
           SUM(output_tokens),
           SUM(cache_creation_input_tokens),
           SUM(cache_read_input_tokens)
    FROM usage
    WHERE DATE(timestamp) = ?
    GROUP BY model
""", (today,))
print("Today's usage:", cursor.fetchall())

# Last 7 days cost estimate (sonnet only)
week_ago = (date.today() - timedelta(days=7)).isoformat()
cursor.execute("""
    SELECT
        SUM(input_tokens) / 1_000_000.0 * 3.69 +
        SUM(output_tokens) / 1_000_000.0 * 18.45 +
        SUM(cache_creation_input_tokens) / 1_000_000.0 * 4.61 +
        SUM(cache_read_input_tokens) / 1_000_000.0 * 0.37 AS estimated_cost
    FROM usage
    WHERE model LIKE '%sonnet%'
      AND DATE(timestamp) >= ?
""", (week_ago,))
cost = cursor.fetchone()[0]
print(f"Estimated sonnet cost last 7 days: ${cost:.4f}")

# Sessions with most tokens
cursor.execute("""
    SELECT session_id, SUM(input_tokens + output_tokens) as total_tokens
    FROM usage
    GROUP BY session_id
    ORDER BY total_tokens DESC
    LIMIT 10
""")
print("Top sessions:", cursor.fetchall())

conn.close()
```

---

## Common Patterns

### Automate daily scan via cron (macOS/Linux)

```bash
# Run scan every hour, log output
crontab -e
# Add:
0 * * * * cd /path/to/claude-usage && python3 cli.py scan >> ~/claude-usage-scan.log 2>&1
```

### Check if the database has been populated

```python
import sqlite3
from pathlib import Path

db_path = Path.home() / ".claude" / "usage.db"
if not db_path.exists():
    print("Database not found — run: python3 cli.py scan")
else:
    conn = sqlite3.connect(db_path)
    count = conn.execute("SELECT COUNT(*) FROM usage").fetchone()[0]
    print(f"Database has {count} usage records")
    conn.close()
```

### Filter dashboard by model via URL

The dashboard supports bookmarkable model filter URLs:

```
http://localhost:8080/?model=sonnet
http://localhost:8080/?model=opus
http://localhost:8080/?model=haiku
```

### Get per-project breakdown

```python
import sqlite3
from pathlib import Path

conn = sqlite3.connect(Path.home() / ".claude" / "usage.db")
cursor = conn.cursor()

cursor.execute("""
    SELECT project,
           COUNT(DISTINCT session_id) as sessions,
           SUM(input_tokens + output_tokens) as total_tokens
    FROM usage
    GROUP BY project
    ORDER BY total_tokens DESC
""")

for project, sessions, tokens in cursor.fetchall():
    print(f"{project}: {sessions} sessions, {tokens:,} tokens")

conn.close()
```

---

## Troubleshooting

### No data appears after scanning

```bash
# Check that Claude Code logs exist
ls ~/.claude/projects/

# Verify the database was created
ls ~/.claude/usage.db

# Run scan with verbose output
python3 cli.py scan
```

### Dashboard shows blank charts

- Run `python3 cli.py scan` first to populate the database
- Dashboard requires internet access for Chart.js CDN — check network connectivity
- Dashboard auto-refreshes every 30 seconds; wait or reload manually

### Python version issues

```bash
python3 --version  # needs 3.8+
# If below 3.8, upgrade Python via pyenv, homebrew, or system package manager
```

### Port 8080 already in use

```python
# Edit dashboard.py or start server on a different port
from dashboard import DashboardServer
DashboardServer(port=8081).serve()
```

### Usage from VS Code extension not showing

Confirm you're using the **Claude Code** VS Code extension (not Claude.ai web). The extension writes to the same `~/.claude/projects/` directory. Re-run `python3 cli.py scan` after using VS Code.

### Cowork session data missing

Cowork sessions run server-side and do not write local JSONL transcripts — this is a platform limitation, not a bug in the tool.

---

## Database Schema Reference

```sql
-- Main usage table (created by scanner.py)
CREATE TABLE IF NOT EXISTS usage (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT,
    session_id TEXT,
    project TEXT,
    model TEXT,
    input_tokens INTEGER,
    output_tokens INTEGER,
    cache_creation_input_tokens INTEGER,
    cache_read_input_tokens INTEGER
);

-- File tracking table (incremental scan state)
CREATE TABLE IF NOT EXISTS scanned_files (
    path TEXT PRIMARY KEY,
    mtime REAL,
    last_scanned TEXT
);
```
```
