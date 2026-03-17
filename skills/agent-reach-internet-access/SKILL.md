```markdown
---
name: agent-reach-internet-access
description: Give AI agents internet access to Twitter, Reddit, YouTube, GitHub, Bilibili, XiaoHongShu and more — one CLI, zero API fees.
triggers:
  - give my agent internet access
  - install agent reach
  - help my AI search twitter reddit youtube
  - set up web scraping for my agent
  - agent reach setup and configuration
  - read tweets without paying for API
  - scrape social media for my AI agent
  - configure agent reach platforms
---

# Agent Reach

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Agent Reach is a scaffolding tool that gives AI coding agents (Claude Code, Cursor, Windsurf, etc.) the ability to read and search the internet — Twitter/X, Reddit, YouTube, GitHub, Bilibili, XiaoHongShu, WeChat, Weibo, RSS, and more — using free upstream CLI tools, zero paid APIs.

## Installation

### One-liner (recommended — tell your agent)

```
帮我安装 Agent Reach：https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md
```

Or in English:

```
Install Agent Reach for me: https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md
```

### Manual install

```bash
pip install agent-reach
```

### Safe mode (no auto system installs)

```
Install Agent Reach (safe mode): https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md
Use the --safe flag
```

### Update

```
帮我更新 Agent Reach：https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/update.md
```

---

## What the installer does

1. Runs `pip install agent-reach`
2. Installs system deps: Node.js, `gh` CLI, `mcporter`, `xreach`
3. Configures Exa search engine via MCP (free, no API key needed)
4. Detects environment (local vs server) and gives proxy advice
5. Registers `SKILL.md` in agent's skills directory

---

## Key CLI Commands

```bash
# Check all channel statuses
agent-reach doctor

# Diagnose a specific channel
agent-reach doctor --channel twitter

# List all available channels
agent-reach channels

# Show version
agent-reach --version
```

---

## Platform Support & Status

| Platform | Works Out of Box | Needs Config |
|---|---|---|
| 🌐 Web pages | ✅ Jina Reader | — |
| 📺 YouTube | ✅ subtitles + search | — |
| 📡 RSS | ✅ feedparser | — |
| 🔍 Web search | ✅ Exa via MCP | — |
| 📦 GitHub | ✅ public repos | login for private |
| 🐦 Twitter/X | ✅ single tweets | cookie for search/timeline |
| 📺 Bilibili | ✅ local machine | proxy for server |
| 📖 Reddit | ✅ search via Exa | proxy for full posts |
| 📕 XiaoHongShu | ❌ | cookie config required |
| 🎵 Douyin | ❌ | MCP config required |
| 💼 LinkedIn | ✅ public pages | MCP for full access |
| 💬 WeChat 公众号 | ✅ search + full article | — |
| 📰 Weibo | ✅ trending, search, user | — |
| 💻 V2EX | ✅ posts, nodes, users | — |
| 🎙️ 小宇宙 Podcast | ❌ | Whisper key required |

---

## How Agents Use It (no commands to memorize)

After install, the agent reads `SKILL.md` and knows which upstream tool to call:

```bash
# Read any webpage
curl https://r.jina.ai/https://example.com

# Read a tweet
xreach tweet https://x.com/user/status/123 --json

# Extract YouTube subtitles
yt-dlp --skip-download --write-auto-sub --sub-format json3 "https://youtube.com/watch?v=VIDEO_ID"

# Search YouTube
yt-dlp "ytsearch5:LLM framework comparison 2025" --dump-json

# Read GitHub repo
gh repo view owner/repo-name

# Search GitHub
gh search repos "LLM framework" --language python --sort stars

# Search GitHub issues
gh search issues "bug label:help-wanted" --repo owner/repo

# Read RSS feed
python -c "import feedparser; f = feedparser.parse('https://example.com/feed.xml'); [print(e.title, e.link) for e in f.entries[:5]]"

# Search web (via Exa MCP — agent calls this automatically)
# No direct CLI; agent invokes via MCP tool call
```

---

## Configuration Workflows

### Twitter/X Cookie Setup

```bash
# 1. Install Chrome extension: Cookie-Editor
# 2. Log into twitter.com in browser
# 3. Click Cookie-Editor → Export → Copy JSON
# 4. Tell your agent: "帮我配 Twitter" and paste the cookie JSON
# Agent will save it to the correct location for xreach
```

Or tell agent: `"帮我配 Twitter"` — it will guide you step by step.

### GitHub Authentication

```bash
gh auth login
# Follow interactive prompts: GitHub.com → HTTPS → browser auth
```

Or tell agent: `"帮我登录 GitHub"`

### XiaoHongShu Setup

```bash
# Tell agent: "帮我配小红书"
# Agent will prompt for:
#   1. Cookie-Editor export from xiaohongshu.com
#   2. Starts xiaohongshu-mcp Docker container
```

Docker-based XHS MCP server:

```bash
docker run -d \
  -e XHS_COOKIE="$XHS_COOKIE" \
  -p 8080:8080 \
  ghcr.io/xpzouying/xiaohongshu-mcp:latest
```

### Proxy Setup (for servers)

```bash
# Tell agent: "帮我配代理"
# Needed only on cloud servers (~$1/month), NOT on local machines
# Agent will configure HTTP_PROXY / HTTPS_PROXY env vars
export HTTP_PROXY=http://your-proxy:port
export HTTPS_PROXY=http://your-proxy:port
```

### Podcast Transcription (小宇宙)

```bash
# Requires free Whisper API key
# Tell agent: "帮我配小宇宙播客"
export OPENAI_API_KEY=your_whisper_key
```

---

## Channel Architecture

Each platform is a pluggable module in `channels/`:

```
channels/
├── web.py          → Jina Reader (r.jina.ai)
├── twitter.py      → xreach CLI
├── youtube.py      → yt-dlp
├── github.py       → gh CLI
├── bilibili.py     → yt-dlp
├── reddit.py       → JSON API + Exa MCP
├── xiaohongshu.py  → xiaohongshu-mcp (Docker)
├── douyin.py       → douyin-mcp-server
├── linkedin.py     → linkedin-mcp-server
├── wechat.py       → camoufox + miku_ai
├── weibo.py        → direct scraping
├── v2ex.py         → V2EX public API
├── rss.py          → feedparser
├── exa_search.py   → mcporter MCP
└── __init__.py     → channel registry
```

Each channel implements a `check()` method used by `agent-reach doctor`.

### Custom channel example

```python
# channels/hackernews.py
from agent_reach.base import BaseChannel

class HackerNewsChannel(BaseChannel):
    name = "hackernews"
    description = "Read Hacker News stories and comments"

    def check(self) -> dict:
        """Returns status dict for agent-reach doctor."""
        try:
            import urllib.request
            urllib.request.urlopen("https://hacker-news.firebaseio.com/v0/topstories.json", timeout=5)
            return {"status": "ok", "message": "HN API reachable"}
        except Exception as e:
            return {"status": "error", "message": str(e)}

    def get_top_stories(self, limit: int = 10) -> list:
        import urllib.request, json
        url = "https://hacker-news.firebaseio.com/v0/topstories.json"
        with urllib.request.urlopen(url) as r:
            ids = json.loads(r.read())[:limit]
        stories = []
        for sid in ids:
            with urllib.request.urlopen(f"https://hacker-news.firebaseio.com/v0/item/{sid}.json") as r:
                stories.append(json.loads(r.read()))
        return stories
```

---

## Real Usage Patterns for Agents

### Pattern 1: Research a topic across platforms

```python
# Agent workflow for "research LLM frameworks":

# 1. Web search
# → MCP Exa tool call: search("best LLM frameworks 2025")

# 2. GitHub trending
import subprocess
result = subprocess.run(
    ["gh", "search", "repos", "LLM framework", "--sort", "stars", "--limit", "10", "--json", "name,description,stargazersCount,url"],
    capture_output=True, text=True
)
import json
repos = json.loads(result.stdout)

# 3. YouTube tutorials
result = subprocess.run(
    ["yt-dlp", "ytsearch5:LLM framework tutorial 2025", "--dump-json", "--flat-playlist"],
    capture_output=True, text=True
)

# 4. Reddit discussion
# → MCP Exa tool call: search("LLM framework site:reddit.com")

# 5. Read a specific article
import urllib.request
article = urllib.request.urlopen("https://r.jina.ai/https://example.com/llm-article").read().decode()
```

### Pattern 2: Monitor Twitter for a topic

```bash
# Search recent tweets
xreach search "LLM framework" --limit 20 --json

# Get a user's timeline
xreach timeline @username --limit 50 --json

# Read a specific tweet thread
xreach tweet https://x.com/user/status/TWEET_ID --json --thread
```

### Pattern 3: YouTube video summarization

```bash
# Extract subtitles for summarization
yt-dlp \
  --skip-download \
  --write-auto-sub \
  --sub-lang en \
  --sub-format vtt \
  --output "/tmp/%(id)s.%(ext)s" \
  "https://youtube.com/watch?v=VIDEO_ID"

# Get video metadata
yt-dlp --dump-json "https://youtube.com/watch?v=VIDEO_ID"
```

```python
# Parse VTT subtitles into clean text
import re

def vtt_to_text(vtt_path: str) -> str:
    with open(vtt_path) as f:
        content = f.read()
    # Remove VTT headers and timestamps
    lines = content.split('\n')
    text_lines = []
    for line in lines:
        line = line.strip()
        if not line or line.startswith('WEBVTT') or '-->' in line or re.match(r'^\d+$', line):
            continue
        # Remove HTML tags
        line = re.sub(r'<[^>]+>', '', line)
        if line:
            text_lines.append(line)
    return ' '.join(text_lines)
```

### Pattern 4: Read Weibo trending & posts

```python
import subprocess, json

# Get trending topics
result = subprocess.run(
    ["agent-reach", "weibo", "trending"],
    capture_output=True, text=True
)
trends = json.loads(result.stdout)

# Search Weibo
result = subprocess.run(
    ["agent-reach", "weibo", "search", "AI大模型", "--limit", "20"],
    capture_output=True, text=True
)
```

### Pattern 5: RSS feed aggregation

```python
import feedparser

def read_feed(url: str, limit: int = 10) -> list[dict]:
    feed = feedparser.parse(url)
    return [
        {
            "title": entry.get("title", ""),
            "link": entry.get("link", ""),
            "summary": entry.get("summary", ""),
            "published": entry.get("published", ""),
        }
        for entry in feed.entries[:limit]
    ]

# Usage
hn_feed = read_feed("https://news.ycombinator.com/rss")
arxiv_feed = read_feed("https://export.arxiv.org/rss/cs.AI")
github_trending = read_feed("https://github.com/trending?since=daily")
```

### Pattern 6: WeChat public account articles

```bash
# Search articles
agent-reach wechat search "AI Agent 2025" --limit 10

# Read full article as Markdown
agent-reach wechat read "https://mp.weixin.qq.com/s/ARTICLE_ID"
```

---

## OpenClaw Users: Enable exec First

```bash
# Without this, agent can't run shell commands
openclaw config set tools.profile "coding"

# Or edit ~/.openclaw/openclaw.json:
# { "tools": { "profile": "coding" } }

# Restart gateway
openclaw gateway restart
```

---

## Troubleshooting

### `agent-reach doctor` output interpretation

```bash
agent-reach doctor

# Example output:
# ✅ web        Jina Reader reachable
# ✅ youtube    yt-dlp v2025.x.x installed
# ✅ github     gh CLI authenticated as @username
# ✅ rss        feedparser 6.x installed
# ⚠️  twitter   xreach installed, no cookie configured
# ❌ bilibili   yt-dlp blocked (server IP) — configure proxy
# ❌ reddit     proxy required for full access
# ✅ search     Exa MCP connected via mcporter
```

### Common issues

**Twitter returns empty results**
```bash
# Cookie expired — re-export from browser
# 1. Open twitter.com, log in
# 2. Cookie-Editor → Export All → JSON
# 3. Tell agent: "帮我配 Twitter" and paste new cookies
xreach config --cookie-file ~/.xreach/cookies.json
```

**Bilibili blocked on server**
```bash
# Set proxy environment variables
export HTTP_PROXY=http://proxy-host:port
export HTTPS_PROXY=http://proxy-host:port
# Test
yt-dlp --dump-json "https://www.bilibili.com/video/BV_ID"
```

**yt-dlp outdated (YouTube changes frequently)**
```bash
pip install -U yt-dlp
# or
yt-dlp -U
```

**gh CLI not authenticated**
```bash
gh auth login
gh auth status  # verify
```

**mcporter / Exa MCP not connecting**
```bash
# Reinstall mcporter
npm install -g mcporter
# Check Node version (requires 18+)
node --version
# Restart MCP server
mcporter restart exa
```

**XiaoHongShu Docker container down**
```bash
docker ps | grep xiaohongshu
docker restart xiaohongshu-mcp
# Or re-run with updated cookie:
docker run -d \
  -e XHS_COOKIE="$XHS_COOKIE" \
  -p 8080:8080 \
  ghcr.io/xpzouying/xiaohongshu-mcp:latest
```

**Jina Reader returns garbled content**
```bash
# Try with Accept header
curl -H "Accept: text/plain" https://r.jina.ai/https://target-url.com
# Or use markdown mode
curl -H "X-Return-Format: markdown" https://r.jina.ai/https://target-url.com
```

---

## Environment Variables Reference

```bash
# Proxy (server deployments only)
HTTP_PROXY=http://host:port
HTTPS_PROXY=http://host:port
NO_PROXY=localhost,127.0.0.1

# XiaoHongShu
XHS_COOKIE=<exported cookie JSON string>

# Douyin MCP
DOUYIN_COOKIE=<exported cookie JSON string>

# LinkedIn MCP
LINKEDIN_EMAIL=your@email.com
LINKEDIN_PASSWORD=$LINKEDIN_PASSWORD  # use secret manager

# Podcast transcription (Whisper)
OPENAI_API_KEY=<your key>

# Agent Reach config dir (default: ~/.agent-reach/)
AGENT_REACH_CONFIG_DIR=/custom/path
```

---

## Project Structure

```
agent-reach/
├── agent_reach/
│   ├── cli.py          # CLI entry point (agent-reach command)
│   ├── doctor.py       # Health check logic
│   ├── base.py         # BaseChannel interface
│   └── channels/       # One file per platform
├── docs/
│   ├── install.md      # Agent-readable install instructions
│   ├── update.md       # Agent-readable update instructions
│   └── README_en.md    # English README
├── skills/
│   └── SKILL.md        # Installed to agent's skills directory
└── pyproject.toml
```

---

## Quick Reference Card

| Goal | Command/Method |
|---|---|
| Health check | `agent-reach doctor` |
| Read webpage | `curl https://r.jina.ai/URL` |
| Read tweet | `xreach tweet URL --json` |
| Search tweets | `xreach search "query" --json` |
| YouTube subtitles | `yt-dlp --write-auto-sub URL` |
| YouTube search | `yt-dlp "ytsearch5:query" --dump-json` |
| GitHub repo | `gh repo view owner/repo` |
| GitHub search | `gh search repos "query"` |
| Read RSS | `feedparser.parse(url)` |
| Web search | Exa via MCP (automatic) |
| Update all tools | `agent-reach update` |
| Configure platform | Tell agent: `"帮我配 [platform]"` |
```
