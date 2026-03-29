```markdown
---
name: last30days-research-skill
description: AI agent skill for the /last30days tool that researches any topic across Reddit, X, Bluesky, YouTube, TikTok, Instagram, Hacker News, Polymarket, and the web from the last 30 days, then synthesizes a grounded summary with real citations.
triggers:
  - research what people are saying about a topic lately
  - find trending discussions on reddit and twitter about something
  - use last30days to look up recent community sentiment
  - search social media and web for recent posts about a topic
  - run last30days skill on a subject
  - find what's trending on hacker news polymarket and youtube
  - get a grounded summary of recent discussions about something
  - install and use the last30days research agent skill
---

# /last30days Research Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`/last30days` is an AI agent skill that researches any topic across Reddit, X (Twitter), Bluesky, YouTube, TikTok, Instagram, Hacker News, Polymarket, and the web — scoped to the last 30 days — and synthesizes a grounded narrative with real citations, engagement scores, and cross-platform convergence signals. It's designed for AI coding agents (Claude Code, Cursor, Codex) but runs anywhere Python + Node.js is available.

---

## Installation

### Claude Code Plugin (Recommended)

```bash
/plugin marketplace add mvanhorn/last30days-skill
/plugin install last30days@last30days-skill
```

### ClawHub

```bash
clawhub install last30days-official
```

### Gemini CLI

```bash
gemini extensions install https://github.com/mvanhorn/last30days-skill.git
```

### Manual Install (Claude Code / Codex CLI)

```bash
# Clone into Claude Code skills directory
git clone https://github.com/mvanhorn/last30days-skill.git ~/.claude/skills/last30days

# For Codex CLI, clone here instead:
git clone https://github.com/mvanhorn/last30days-skill.git ~/.agents/skills/last30days
```

---

## Configuration

Create the global config file at `~/.config/last30days/.env`:

```bash
mkdir -p ~/.config/last30days
cat > ~/.config/last30days/.env << 'EOF'
# Reddit + TikTok + Instagram — one key covers all three
SCRAPECREATORS_API_KEY=your_scrapecreators_key   # scrapecreators.com

# X (Twitter) — cookie-based auth (recommended)
AUTH_TOKEN=your_x_auth_token     # copy from x.com browser cookies
CT0=your_x_ct0_token             # copy from x.com browser cookies

# X fallback — xAI backend if you don't want cookie auth
XAI_API_KEY=your_xai_key

# Bluesky (optional)
BSKY_HANDLE=you.bsky.social
BSKY_APP_PASSWORD=xxxx-xxxx-xxxx   # create at bsky.app/settings/app-passwords

# Web search backends (optional, open variant)
PARALLEL_API_KEY=your_parallel_key      # Parallel AI (preferred)
BRAVE_API_KEY=your_brave_key            # Brave Search (2,000 free queries/month)
OPENROUTER_API_KEY=your_openrouter_key  # OpenRouter / Perplexity Sonar Pro

# OpenAI (optional — skip if using `codex login`)
OPENAI_API_KEY=sk-your_openai_key
EOF
chmod 600 ~/.config/last30days/.env
```

### Per-Project Override

Drop a `.claude/last30days.env` file in your project root to override global keys for a specific repo:

```bash
# .claude/last30days.env
SCRAPECREATORS_API_KEY=project_specific_key
BSKY_HANDLE=project-account.bsky.social
```

### Requirements

- **Python 3.10+**
- **Node.js 22+** (for the vendored X/Twitter GraphQL client)

---

## Usage

### Basic Syntax

```
/last30days [topic]
/last30days [topic] for [tool]
/last30days [topic A] vs [topic B]
```

### Flags

| Flag | Description |
|------|-------------|
| `--quick` | Faster, shallower search — fewer sources, no transcript fetch |
| `--days=N` | Change the lookback window (default: 30) |
| `--diagnose` | Check which sources are available with current API keys |

### Examples

```bash
# Prompt research for a specific tool
/last30days prompting techniques for ChatGPT for legal questions

# Tool trend discovery
/last30days remotion animations for Claude Code

# Comparative mode (runs 3 parallel research passes + side-by-side verdict)
/last30days cursor vs windsurf

# Cultural / trend queries
/last30days what are the best rap songs lately
/last30days dog-as-human trend on ChatGPT

# Product research
/last30days what do people think of the new M4 MacBook

# Prediction market signals
/last30days Arizona basketball NCAA tournament odds
```

---

## What the Skill Does (Pipeline)

```
User query
    │
    ├─► Query expansion + synonym normalization
    ├─► X handle resolution (e.g. "Dor Brothers" → @thedorbrothers)
    │
    ├─► Parallel source fetch (up to 10 sources simultaneously):
    │       Reddit (ScrapeCreators)
    │       X / Twitter (cookie auth or xAI fallback)
    │       Bluesky / AT Protocol
    │       YouTube (with transcript extraction)
    │       TikTok (ScrapeCreators)
    │       Instagram Reels (ScrapeCreators)
    │       Hacker News (stories + comments)
    │       Polymarket (prediction markets)
    │       Web (Parallel AI / Brave / OpenRouter)
    │
    ├─► Composite scoring per result:
    │       - Bidirectional text similarity + synonym expansion
    │       - Engagement velocity normalization
    │       - Source authority weighting
    │       - Cross-platform convergence (trigram-token Jaccard)
    │       - Temporal recency decay
    │       - Polymarket: volume (30%) + liquidity (15%) + price movement (15%)
    │           + competitiveness (10%) + text relevance (30%)
    │
    ├─► Deduplication
    ├─► Top comment elevation (10% scoring weight, 💬 display)
    │
    └─► Synthesis → grounded narrative with citations
            └─► Auto-saved to ~/Documents/Last30Days/[topic].md
```

---

## Key Commands (Python Engine)

### Diagnose Source Availability

```bash
python3 ~/.claude/skills/last30days/scripts/last30days.py --diagnose
```

Sample output:
```
✅ Reddit       (SCRAPECREATORS_API_KEY set)
✅ TikTok       (SCRAPECREATORS_API_KEY set)
✅ Instagram    (SCRAPECREATORS_API_KEY set)
✅ X            (AUTH_TOKEN + CT0 set)
⚠️  Bluesky     (BSKY_HANDLE or BSKY_APP_PASSWORD missing)
✅ YouTube      (no auth needed)
✅ Hacker News  (no auth needed)
⚠️  Polymarket  (no auth needed, but network unavailable)
⚠️  Web search  (no PARALLEL_API_KEY, BRAVE_API_KEY, or OPENROUTER_API_KEY)
```

### Verify X Authentication

```bash
node ~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs --whoami
```

### Run a One-Off Research Query Directly

```bash
python3 ~/.claude/skills/last30days/scripts/last30days.py "your topic here"
python3 ~/.claude/skills/last30days/scripts/last30days.py "cursor vs windsurf" --days=14
python3 ~/.claude/skills/last30days/scripts/last30days.py "AI video tools" --quick
```

---

## Watchlist + Briefings (Open Variant)

The open variant is designed for always-on bots ([Open Claw](https://github.com/openclaw/openclaw)) or cron-scheduled research pipelines.

### Enable Open Variant

```bash
cp ~/.claude/skills/last30days/variants/open/SKILL.md \
   ~/.claude/skills/last30days/SKILL.md
```

### Watchlist Commands

```bash
# Add topics to watch
last30 watch my biggest competitor every week
last30 watch Peter Steinberger every 30 days
last30 watch AI video tools monthly
last30 watch Y Combinator hot companies end of April and end of September

# Run research manually
last30 run all my watched topics
last30 run one "AI video tools"

# Query accumulated findings
last30 what have you found about AI video?
last30 briefing for this week
```

> **Note:** The watchlist stores schedules as metadata but does **not** auto-trigger runs. Use `cron`, `launchd`, or an always-on bot to call `watchlist.py run-all` on a timer.

### Cron Example

```bash
# Research all watched topics nightly at 2am
0 2 * * * python3 ~/.claude/skills/last30days/scripts/watchlist.py run-all
```

### Storage

Accumulated findings are stored in a local SQLite database. Full-text search is supported:

```bash
# Direct DB query (advanced)
python3 ~/.claude/skills/last30days/scripts/history.py search "AI agents"
python3 ~/.claude/skills/last30days/scripts/history.py briefing --days=7
```

---

## Code Examples

### Call the Python Engine Directly

```python
import subprocess
import json

def research_topic(topic: str, days: int = 30, quick: bool = False) -> str:
    """Run a last30days research query and return the synthesized output."""
    cmd = [
        "python3",
        f"{Path.home()}/.claude/skills/last30days/scripts/last30days.py",
        topic,
        f"--days={days}",
    ]
    if quick:
        cmd.append("--quick")

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=600)
    if result.returncode != 0:
        raise RuntimeError(f"last30days error: {result.stderr}")
    return result.stdout

# Example usage
summary = research_topic("cursor vs windsurf", days=14)
print(summary)
```

### Load a Saved Briefing

```python
from pathlib import Path

def load_briefing(topic_slug: str) -> str:
    """Load a previously auto-saved last30days briefing."""
    docs_dir = Path.home() / "Documents" / "Last30Days"
    # Files are saved as sanitized topic names
    candidates = list(docs_dir.glob(f"*{topic_slug}*.md"))
    if not candidates:
        raise FileNotFoundError(f"No briefing found for '{topic_slug}'")
    # Most recently modified first
    latest = sorted(candidates, key=lambda p: p.stat().st_mtime, reverse=True)[0]
    return latest.read_text()

briefing = load_briefing("cursor-vs-windsurf")
print(briefing[:500])
```

### Check API Key Configuration

```python
import os
from pathlib import Path
from dotenv import dotenv_values

def check_last30_config() -> dict[str, bool]:
    """Check which last30days sources are configured."""
    global_env = Path.home() / ".config" / "last30days" / ".env"
    project_env = Path(".claude/last30days.env")

    config = {}
    if global_env.exists():
        config.update(dotenv_values(global_env))
    if project_env.exists():
        config.update(dotenv_values(project_env))  # project overrides global

    return {
        "reddit_tiktok_instagram": bool(config.get("SCRAPECREATORS_API_KEY")),
        "x_twitter_cookies":       bool(config.get("AUTH_TOKEN") and config.get("CT0")),
        "x_twitter_xai":           bool(config.get("XAI_API_KEY")),
        "bluesky":                 bool(config.get("BSKY_HANDLE") and config.get("BSKY_APP_PASSWORD")),
        "web_parallel":            bool(config.get("PARALLEL_API_KEY")),
        "web_brave":               bool(config.get("BRAVE_API_KEY")),
        "web_openrouter":          bool(config.get("OPENROUTER_API_KEY")),
        "youtube":                 True,   # no auth needed
        "hackernews":              True,   # no auth needed
        "polymarket":              True,   # no auth needed
    }

status = check_last30_config()
for source, available in status.items():
    icon = "✅" if available else "⚠️ "
    print(f"{icon} {source}")
```

---

## Output Format

Every run produces a structured briefing:

```markdown
## /last30days: [Topic] — Last 30 Days

### 🔥 What's Trending
Narrative synthesis of the top signals across sources...

### 📊 Source Breakdown

**Reddit** (r/AIAssistants, r/cursor)
- [Post title](url) — 847 upvotes · 92 comments
  💬 Top comment: "The autocomplete latency difference is night and day..." (312 upvotes)

**X / Twitter**
- [@handle](url): "Quote from viral tweet..." — 5,600 likes

**Hacker News**
- [Story title](url) — 234 points · 89 comments

**Polymarket**
- [Market: Will Cursor reach 1M users by Q2?](url) — Yes: 67% · Volume: $142k/24h

**YouTube**
- [Video title](url) — 48k views · Key insight from transcript: "..."

### 🏆 Cross-Platform Consensus
Topics that appeared in 3+ sources simultaneously...

### 📁 Saved to
~/Documents/Last30Days/cursor-vs-windsurf-2026-03-29.md
```

---

## Comparative Mode

Append `vs` between two topics to trigger 3 parallel research passes and a side-by-side verdict:

```bash
/last30days cursor vs windsurf
/last30days claude vs gpt-4o for coding
/last30days react vs svelte 2026
```

Output includes:
- Individual strengths and weaknesses per option
- Head-to-head comparison table (performance, community sentiment, pricing, HN score, prediction market signals)
- A data-driven verdict with caveats

---

## Troubleshooting

### X Search Returns No Results

```bash
# Verify cookie auth is working
node ~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs --whoami

# If it fails, re-copy fresh cookies from x.com browser dev tools:
# Application → Cookies → x.com → auth_token and ct0
# Update AUTH_TOKEN and CT0 in ~/.config/last30days/.env
```

### Reddit Returns No Results

```bash
# Confirm ScrapeCreators key is valid
python3 -c "
import os; from pathlib import Path; from dotenv import dotenv_values
cfg = dotenv_values(Path.home() / '.config/last30days/.env')
print('SCRAPECREATORS_API_KEY set:', bool(cfg.get('SCRAPECREATORS_API_KEY')))
"
```

### Skill Takes Too Long

```bash
# Use --quick for speed over thoroughness (skips transcript fetch, fewer sources)
/last30days your topic --quick

# Or narrow the time window
/last30days your topic --days=7
```

### Bluesky Not Searching

Ensure you've created an **App Password** (not your account password):
1. Go to `bsky.app/settings/app-passwords`
2. Create a new app password
3. Set `BSKY_HANDLE` and `BSKY_APP_PASSWORD` in your `.env`

### Session Config Check Fails at Startup

The skill validates config on `SessionStart`. Run diagnose to see exactly what's missing:

```bash
python3 ~/.claude/skills/last30days/scripts/last30days.py --diagnose
```

### Auto-Save Not Working

Check that `~/Documents/Last30Days/` exists and is writable:

```bash
mkdir -p ~/Documents/Last30Days
ls -la ~/Documents/Last30Days/
```

---

## Source Priority & Free Tier

| Source | Auth Required | Free Tier |
|--------|--------------|-----------|
| YouTube | None | ✅ Always available |
| Hacker News | None | ✅ Always available |
| Polymarket | None | ✅ Always available |
| Reddit | `SCRAPECREATORS_API_KEY` | Paid |
| TikTok | `SCRAPECREATORS_API_KEY` | Paid (same key as Reddit) |
| Instagram | `SCRAPECREATORS_API_KEY` | Paid (same key as Reddit) |
| X (Twitter) | `AUTH_TOKEN` + `CT0` or `XAI_API_KEY` | Cookies are free; xAI is paid |
| Bluesky | `BSKY_HANDLE` + `BSKY_APP_PASSWORD` | ✅ Free |
| Web (Brave) | `BRAVE_API_KEY` | 2,000 queries/month free |
| Web (Parallel AI) | `PARALLEL_API_KEY` | Paid |
| Web (OpenRouter) | `OPENROUTER_API_KEY` | Paid |

**Minimum useful config:** No API keys required — YouTube, HN, and Polymarket work out of the box. Add `SCRAPECREATORS_API_KEY` + X cookies for full coverage.

---

## Version

Current: **v2.9.5** | Tests: 455+ | Avg runtime: 2–8 minutes (30s with `--quick`)
```
