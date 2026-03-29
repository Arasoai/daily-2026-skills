```markdown
---
name: last30days-research-skill
description: AI agent skill for researching any topic across Reddit, X, YouTube, HN, Polymarket, and the web to synthesize grounded, citation-backed summaries from the last 30 days
triggers:
  - research what people are saying about
  - find recent discussions about
  - what's trending in the last 30 days
  - summarize community sentiment on
  - research across reddit and twitter about
  - find what people are actually saying about
  - deep research current opinions on
  - last30days investigate topic
---

# /last30days Research Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`/last30days` is a multi-source research skill that scans Reddit, X (Twitter), Bluesky, YouTube, TikTok, Instagram, Hacker News, Polymarket, and the web for discussions from the last 30 days, scores and deduplicates results across a composite relevance pipeline, and synthesizes a grounded narrative with real citations. One command surfaces what people who are paying attention already know.

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

# For Codex CLI, use the agents directory instead
git clone https://github.com/mvanhorn/last30days-skill.git ~/.agents/skills/last30days
```

---

## Configuration

Create a global config file at `~/.config/last30days/.env`:

```bash
mkdir -p ~/.config/last30days
cat > ~/.config/last30days/.env << 'EOF'
# Reddit + TikTok + Instagram — one key covers all three
SCRAPECREATORS_API_KEY=...

# X (Twitter) cookie-based auth — copy from browser dev tools while logged into x.com
AUTH_TOKEN=...
CT0=...

# xAI fallback if you don't want cookie-based X auth
XAI_API_KEY=...

# Bluesky (optional)
BSKY_HANDLE=you.bsky.social
BSKY_APP_PASSWORD=xxxx-xxxx-xxxx

# Web search backends (optional, open variant)
PARALLEL_API_KEY=...
BRAVE_API_KEY=...
OPENROUTER_API_KEY=...

# OpenAI (optional legacy Reddit fallback)
OPENAI_API_KEY=...
EOF
chmod 600 ~/.config/last30days/.env
```

### Per-Project Config Override

Drop a `.claude/last30days.env` in any project root. It overrides the global config for that project only:

```bash
# .claude/last30days.env
SCRAPECREATORS_API_KEY=...
AUTH_TOKEN=...
CT0=...
```

### Setting Up X Authentication

1. Log into x.com in your browser
2. Open Dev Tools → Application → Cookies → `x.com`
3. Copy `auth_token` → save as `AUTH_TOKEN`
4. Copy `ct0` → save as `CT0`
5. Verify it works:

```bash
node ~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs --whoami
```

> **Note:** Node.js 22+ is required for the bundled Twitter GraphQL client.

### Setting Up Bluesky

1. Go to `bsky.app/settings/app-passwords`
2. Create an app password
3. Add `BSKY_HANDLE` and `BSKY_APP_PASSWORD` to your `.env`

---

## Usage

### Basic Syntax

```
/last30days [topic]
/last30days [topic] for [tool]
```

### Example Queries

```bash
# Prompt research for a specific tool
/last30days prompting techniques for ChatGPT for legal questions

# Tool best practices
/last30days remotion animations for Claude Code

# Trend discovery
/last30days what are the best rap songs lately

# Product research
/last30days what do people think of the new M4 MacBook

# Comparative mode (v2.9.5+)
/last30days cursor vs windsurf

# Handle resolution — finds @thedorbrothers and searches their posts
/last30days Dor Brothers

# Scoped time window
/last30days AI video tools --days=7

# Quick mode (speed over thoroughness)
/last30days AI video tools --quick

# Diagnose which sources are configured
python3 ~/.claude/skills/last30days/scripts/last30days.py --diagnose
```

---

## Key Features

### Multi-Source Research Pipeline

Sources searched in parallel, results scored and deduplicated:

| Source | Auth Required | Notes |
|---|---|---|
| Reddit | `SCRAPECREATORS_API_KEY` | Smart subreddit discovery, top comments scored |
| X / Twitter | `AUTH_TOKEN` + `CT0` or `XAI_API_KEY` | Handle resolution for people/brands |
| Bluesky | `BSKY_HANDLE` + `BSKY_APP_PASSWORD` | Full AT Protocol pipeline |
| YouTube | None (transcripts) | Video transcripts as signal source |
| TikTok | `SCRAPECREATORS_API_KEY` | Same key as Reddit + Instagram |
| Instagram Reels | `SCRAPECREATORS_API_KEY` | Same key as Reddit + TikTok |
| Hacker News | None | Stories, Show HN, comment insights |
| Polymarket | None | Real-money prediction markets |
| Web | Optional (`PARALLEL_API_KEY`, `BRAVE_API_KEY`, etc.) | Open variant |

### Composite Relevance Scoring

Every result runs through:
- Bidirectional text similarity with synonym expansion and token overlap
- Engagement velocity normalization
- Source authority weighting
- Cross-platform convergence detection (hybrid trigram-token Jaccard similarity)
- Temporal recency decay

Polymarket markets use a 5-factor weighted composite:
- Text relevance (30%)
- 24-hour volume (30%)
- Liquidity depth (15%)
- Price movement velocity (15%)
- Outcome competitiveness (10%)

### Comparative Mode

Ask `X vs Y` to trigger 3 parallel research passes with a side-by-side comparison:

```bash
/last30days cursor vs windsurf
# Returns: strengths, weaknesses, head-to-head table, data-driven verdict
```

### Auto-Save to Documents

Every run saves a `.md` briefing to `~/Documents/Last30Days/[topic].md` automatically — builds a personal research library over time.

---

## Watchlist / Open Variant

For always-on bots and scheduled research (designed for [Open Claw](https://github.com/openclaw/openclaw)):

```bash
# Switch to open variant
cp ~/.claude/skills/last30days/variants/open/SKILL.md ~/.claude/skills/last30days/SKILL.md
```

### Watchlist Commands

```bash
# Add topics to watchlist
last30 watch my biggest competitor every week
last30 watch Peter Steinberger every 30 days
last30 watch AI video tools monthly

# Run research manually (or wire to cron)
last30 run all my watched topics
last30 run one "AI video tools"

# Query accumulated knowledge
last30 what have you found about AI video?
```

### Cron Integration

```bash
# Run all watched topics daily at 8am
0 8 * * * python3 ~/.claude/skills/last30days/scripts/last30days.py watchlist run-all
```

> **Important:** The watchlist stores schedules as metadata but does NOT trigger runs automatically. You need an external scheduler (cron, launchd, or an always-on bot) to call `watchlist.py run-all` on a timer.

---

## Python Engine — Direct Usage

```python
import subprocess
import json

# Run a research query programmatically
result = subprocess.run(
    ["python3", "~/.claude/skills/last30days/scripts/last30days.py",
     "--topic", "Claude Code AI agents",
     "--days", "30",
     "--format", "json"],
    capture_output=True,
    text=True
)

data = json.loads(result.stdout)
print(data["summary"])
print(data["sources"])
```

### Diagnose Source Availability

```python
import subprocess

result = subprocess.run(
    ["python3", "~/.claude/skills/last30days/scripts/last30days.py", "--diagnose"],
    capture_output=True,
    text=True
)
print(result.stdout)
# Output: which sources are configured and available
```

### Check Bluesky Auth

```python
import subprocess

result = subprocess.run(
    ["python3", "~/.claude/skills/last30days/scripts/lib/bluesky.py", "--check"],
    capture_output=True,
    text=True
)
print(result.stdout)
```

---

## Common Patterns

### Prompt Research Workflow

Best use case: discover what prompting techniques work for any AI tool by learning from real community discussions.

```bash
# Find what works for image generation
/last30days nano banana pro prompting

# Find legal AI prompting techniques
/last30days prompting techniques for ChatGPT for legal questions

# Find coding agent workflows
/last30days Claude Code agentic workflows best practices
```

### Trend Discovery

```bash
# Music / culture
/last30days best rap songs lately

# Viral AI techniques
/last30days dog as human trend ChatGPT

# Tech product launches
/last30days Seedance 2.0 access
```

### Competitive Intelligence

```bash
# Compare two products
/last30days linear vs jira 2025

# Track a competitor
last30 watch CompetitorName weekly   # open variant

# Find community sentiment on a product
/last30days what do people think of Cursor AI
```

### Prediction Market Research

```bash
# Sports betting odds from Polymarket
/last30days Arizona Basketball

# Geopolitical markets
/last30days Iran War

# Tech market outcomes
/last30days OpenAI GPT-5 release
```

---

## Troubleshooting

### X Search Not Working

```bash
# Verify your cookies are valid
node ~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs --whoami

# If expired, re-copy auth_token and ct0 from browser dev tools at x.com
# Then update ~/.config/last30days/.env
```

### Reddit Returning No Results

```bash
# Confirm SCRAPECREATORS_API_KEY is set
python3 ~/.claude/skills/last30days/scripts/last30days.py --diagnose

# Check key is exported
echo $SCRAPECREATORS_API_KEY
```

### Bluesky Auth Failing

```bash
# Ensure you're using an App Password, NOT your account password
# Create one at: bsky.app/settings/app-passwords

# Verify format in .env:
# BSKY_HANDLE=yourhandle.bsky.social
# BSKY_APP_PASSWORD=xxxx-xxxx-xxxx-xxxx
```

### Node.js Version Error

```bash
# Check version — must be 22+
node --version

# Upgrade via nvm
nvm install 22
nvm use 22
```

### Slow Results

```bash
# Use quick mode for faster (less thorough) results
/last30days [topic] --quick

# Narrow the time window
/last30days [topic] --days=7
```

### Config Not Loading

```bash
# Check global config exists and has correct permissions
ls -la ~/.config/last30days/.env   # should be -rw-------

# Check per-project override
ls -la .claude/last30days.env

# SessionStart config check (v2.9.5+) runs automatically when Claude Code starts
```

---

## Source Summary

- **ScrapeCreators** (`scrapecreators.com`) — one API key covers Reddit, TikTok, Instagram
- **Bird Search** — bundled Node.js Twitter GraphQL client for X, requires `AUTH_TOKEN` + `CT0`
- **xAI** — alternative X backend if you don't want cookie-based auth
- **Bluesky AT Protocol** — native search via app password
- **Polymarket** — public API, no auth required
- **Hacker News** — public Algolia API, no auth required
- **YouTube** — transcript extraction, no auth required for public videos
- **Web Search** — optional: Parallel AI, Brave Search, OpenRouter/Perplexity Sonar Pro

---

## Links

- [GitHub](https://github.com/mvanhorn/last30days-skill)
- [ClawHub listing](https://clawhub.ai/skills/last30days-official)
- [ScrapeCreators](https://scrapecreators.com) — Reddit + TikTok + Instagram API
- [Open Claw](https://github.com/openclaw/openclaw) — always-on bot for watchlist scheduling
```
