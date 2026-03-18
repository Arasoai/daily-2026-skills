---
name: opencli-social-platforms
description: Use opencli CLI to control Bilibili, Twitter/X, YouTube, Zhihu, Reddit, HackerNews, and 10+ other platforms via natural language, reusing your Chrome login sessions with no API keys needed.
triggers:
  - search YouTube for videos
  - get HackerNews top stories
  - check Twitter trending
  - browse Bilibili hot videos
  - search Reddit posts
  - check stock price on Yahoo Finance
  - post a tweet using Claude
  - get Zhihu hot list
---

# opencli-skill: Control 16 Social Platforms via CLI

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

opencli is a CLI tool that turns 16 major social/content platforms into command-line interfaces by **reusing your existing Chrome browser login sessions**. No API keys, no re-authentication — just open Chrome, log in as usual, and your AI agent does the rest.

## Prerequisites

Before using opencli, ensure all of these are in place:

1. **Node.js v16+** — [nodejs.org](https://nodejs.org/)
2. **Chrome browser** open and logged in to target platforms
3. **Playwright MCP Bridge** Chrome extension — [Install from Chrome Web Store](https://chromewebstore.google.com/detail/playwright-mcp-bridge/kldoghpdblpjbjeechcaoibpfbgfomkn)
4. **Playwright MCP** configured in your AI agent (Claude Code, etc.)
5. **opencli** installed globally

## Installation

### Step 1 — Install opencli

```bash
npm install -g @jackwener/opencli

# Verify installation
opencli --version
```

### Step 2 — Install Playwright MCP Bridge in Chrome

1. Open Chrome → go to the [Playwright MCP Bridge extension page](https://chromewebstore.google.com/detail/playwright-mcp-bridge/kldoghpdblpjbjeechcaoibpfbgfomkn)
2. Click **"Add to Chrome"** and confirm
3. Verify the extension icon appears in Chrome's toolbar

### Step 3 — Configure Playwright MCP in Claude Code

```bash
# Add Playwright MCP server (run once)
claude mcp add playwright --scope user -- npx @playwright/mcp@latest

# Verify it was added
claude mcp list
# Should show "playwright" in the list
```

### Step 4 — Install this Skill

```bash
npx skills add joeseesun/opencli-skill
```

Restart Claude Code after installation.

## Supported Platforms

| Platform | Read | Search | Write |
|----------|------|--------|-------|
| Bilibili (B站) | ✅ Hot/Ranking/Feed/History | ✅ Videos/Users | — |
| Zhihu (知乎) | ✅ Hot list | ✅ | ✅ Question details |
| Weibo (微博) | ✅ Trending | — | ✅ Post |
| Twitter/X | ✅ Timeline/Trending/Bookmarks | ✅ | ✅ Post/Reply/Like |
| YouTube | — | ✅ | — |
| Xiaohongshu (小红书) | ✅ Recommended feed | ✅ | — |
| Reddit | ✅ Home/Hot | ✅ | — |
| HackerNews | ✅ Top stories | — | — |
| V2EX | ✅ Hot/Latest | — | ✅ Daily check-in |
| Xueqiu (雪球) | ✅ Hot/Stocks/Watchlist | ✅ | — |
| BOSS直聘 | — | ✅ Jobs | — |
| BBC | ✅ News | — | — |
| Reuters | — | ✅ | — |
| 什么值得买 | — | ✅ Deals | — |
| Yahoo Finance | ✅ Stock quotes | — | — |
| Ctrip (携程) | — | ✅ Attractions/Cities | — |

## Key Commands

### Bilibili (B站)

```bash
# Get hot/trending videos
opencli bilibili hot --limit 10 -f json

# Get video rankings
opencli bilibili ranking -f json

# Get your feed/timeline
opencli bilibili feed -f json

# Get watch history
opencli bilibili history -f json

# Search videos
opencli bilibili search --keyword "AI大模型"

# Search users
opencli bilibili search-user --keyword "技术up主"
```

### Twitter/X

```bash
# Get your timeline
opencli twitter timeline -f json

# Get trending topics
opencli twitter trending -f json

# Get your bookmarks
opencli twitter bookmarks -f json

# Search tweets
opencli twitter search --query "claude AI" -f json

# Post a tweet (REQUIRES CONFIRMATION — content becomes immediately public)
opencli twitter post --text "Hello from Claude Code!"

# Like a tweet
opencli twitter like --id TWEET_ID

# Reply to a tweet
opencli twitter reply --id TWEET_ID --text "Great point!"
```

### YouTube

```bash
# Search videos
opencli youtube search --query "LLM tutorial" -f json

# Search with limit
opencli youtube search --query "machine learning 2024" --limit 20 -f json
```

### Zhihu (知乎)

```bash
# Get hot list
opencli zhihu hot -f json

# Search questions/answers
opencli zhihu search --keyword "大模型" -f json

# Get question details
opencli zhihu question --id QUESTION_ID -f json
```

### Weibo (微博)

```bash
# Get trending hot search
opencli weibo hot -f json

# Post a Weibo update (uses Playwright — REQUIRES CONFIRMATION)
opencli weibo post --text "今天天气真好"
```

### Reddit

```bash
# Get home feed
opencli reddit home -f json

# Get hot posts from a subreddit
opencli reddit hot --subreddit MachineLearning -f json

# Search Reddit
opencli reddit search --query "transformer architecture" -f json
```

### HackerNews

```bash
# Get top stories
opencli hackernews top --limit 20 -f json

# Get top stories (default limit)
opencli hackernews top -f json
```

### Yahoo Finance / Xueqiu (雪球) — Stocks

```bash
# Get stock quote (Yahoo Finance)
opencli yahoo-finance quote --symbol AAPL -f json
opencli yahoo-finance quote --symbol TSLA -f json

# Xueqiu hot stocks
opencli xueqiu hot -f json

# Xueqiu stock details
opencli xueqiu stock --symbol SH600519   # Moutai (茅台)
opencli xueqiu stock --symbol SZ000001   # Ping An Bank

# Xueqiu watchlist (your followed stocks)
opencli xueqiu watchlist -f json

# Search stocks on Xueqiu
opencli xueqiu search --keyword "茅台" -f json
```

### Xiaohongshu (小红书)

```bash
# Get recommended feed
opencli xiaohongshu feed -f json

# Search posts
opencli xiaohongshu search --keyword "咖啡" -f json
```

### V2EX

```bash
# Get hot posts
opencli v2ex hot -f json

# Get latest posts
opencli v2ex latest -f json

# Daily check-in (签到)
opencli v2ex checkin
```

### BOSS直聘 (Job Search)

```bash
# Search jobs
opencli boss search --keyword "Python工程师" -f json
opencli boss search --keyword "AI researcher" --city "北京" -f json
```

### BBC News

```bash
# Get BBC headlines
opencli bbc news -f json
```

### Reuters

```bash
# Search Reuters
opencli reuters search --query "artificial intelligence" -f json
```

### 什么值得买 (Deals)

```bash
# Search deals
opencli smzdm search --keyword "机械键盘" -f json
```

### Ctrip / 携程 (Travel)

```bash
# Search attractions
opencli ctrip attractions --city "成都" -f json

# Search cities
opencli ctrip cities --keyword "云南" -f json
```

## Output Format

Most commands support `-f json` for structured JSON output, which is ideal for AI agents to parse and display:

```bash
# JSON output (recommended for AI processing)
opencli hackernews top --limit 10 -f json

# Example output structure:
# [
#   {
#     "rank": 1,
#     "title": "Show HN: ...",
#     "url": "https://...",
#     "points": 342,
#     "comments": 87
#   },
#   ...
# ]
```

## Common Agent Patterns

### Pattern 1: Research across platforms

```bash
# User asks: "What's trending in AI today?"

# Check HackerNews
opencli hackernews top --limit 10 -f json

# Check Twitter trending
opencli twitter trending -f json

# Search Zhihu for AI discussions
opencli zhihu search --keyword "人工智能" -f json

# Search Reddit
opencli reddit search --query "artificial intelligence" -f json
```

### Pattern 2: Stock research

```bash
# User asks: "How is Tesla doing today?"

# Yahoo Finance for US stocks
opencli yahoo-finance quote --symbol TSLA -f json

# Check Xueqiu for Chinese market sentiment
opencli xueqiu hot -f json
```

### Pattern 3: Content discovery

```bash
# User asks: "Find LLM tutorials on YouTube and Bilibili"

opencli youtube search --query "LLM tutorial 2024" --limit 10 -f json
opencli bilibili search --keyword "大模型教程" -f json
```

### Pattern 4: Job searching

```bash
# User asks: "Find AI engineer jobs in Beijing"

opencli boss search --keyword "AI工程师" --city "北京" -f json
```

### Pattern 5: Write operations (always confirm first)

```bash
# IMPORTANT: Always show the user what will be posted and get explicit confirmation

# Post to Twitter
opencli twitter post --text "Just discovered opencli — control 16 platforms from the terminal!"

# V2EX check-in
opencli v2ex checkin

# Post to Weibo
opencli weibo post --text "今天用 Claude Code 搜了B站热门，太方便了"
```

## ⚠️ Write Operations — Safety Rules

**Always follow these rules for any write operation:**

1. **Show content first** — Display exactly what will be posted to the user
2. **Require explicit confirmation** — Wait for user to say "yes", "confirm", "post it", etc.
3. **Never auto-post** — Even if the user said "post X", confirm the exact content before executing
4. **One operation at a time** — Never batch multiple write operations

Write operations include:
- `opencli twitter post` / `reply` / `like`
- `opencli weibo post`
- `opencli v2ex checkin`
- Any command that modifies platform state

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `opencli: command not found` | Not installed or PATH issue | Run `npm install -g @jackwener/opencli`; check `echo $PATH` includes npm global bin |
| Chrome not being controlled | Extension missing or Chrome closed | Ensure Chrome is open; verify Playwright MCP Bridge extension is enabled in `chrome://extensions` |
| Login state not recognized | Not logged in to the site | Open Chrome, manually log in to the target platform, then retry |
| "Playwright MCP not found" error | MCP not configured | Run `claude mcp add playwright --scope user -- npx @playwright/mcp@latest` |
| `npx skills add` fails | Node.js version too old | Upgrade to Node.js v16+: `node --version` to check |
| Rate limit / CAPTCHA triggered | Too many rapid requests | Wait a few minutes; avoid repeated rapid commands to same platform |
| JSON parse errors | Command output is not JSON | Ensure you're using `-f json` flag; some commands may not support JSON |

### Verify Setup

```bash
# 1. Check opencli is installed
opencli --version

# 2. Check Node.js version
node --version   # Should be v16+

# 3. Check Playwright MCP is configured (in Claude Code)
claude mcp list  # Should show "playwright"

# 4. Test a simple read command
opencli hackernews top --limit 3 -f json
```

### Chrome Extension Verification

1. Open Chrome → navigate to `chrome://extensions/`
2. Find "Playwright MCP Bridge"
3. Confirm it shows **"Enabled"** (toggle is blue)
4. If missing, reinstall from the [Chrome Web Store](https://chromewebstore.google.com/detail/playwright-mcp-bridge/kldoghpdblpjbjeechcaoibpfbgfomkn)

## Natural Language → Command Mapping

When a user makes a request, map it to the appropriate command:

| User Says | Command |
|-----------|---------|
| "What's trending on Twitter?" | `opencli twitter trending -f json` |
| "Search YouTube for X" | `opencli youtube search --query "X" -f json` |
| "Get HackerNews top stories" | `opencli hackernews top --limit 20 -f json` |
| "Check AAPL stock" | `opencli yahoo-finance quote --symbol AAPL -f json` |
| "What's hot on Bilibili?" | `opencli bilibili hot --limit 20 -f json` |
| "Search Reddit for X" | `opencli reddit search --query "X" -f json` |
| "Get Zhihu hot list" | `opencli zhihu hot -f json` |
| "What's trending on Weibo?" | `opencli weibo hot -f json` |
| "Find jobs for X" | `opencli boss search --keyword "X" -f json` |
| "Check my Twitter timeline" | `opencli twitter timeline -f json` |
| "Post a tweet saying X" | Confirm with user → `opencli twitter post --text "X"` |

## Full Command Reference

See [references/commands.md](references/commands.md) in the skill repository for the complete list of all 55 commands with parameters.

## Credits

Built on **[jackwener/opencli](https://github.com/jackwener/opencli)** by [@jakevin7](https://github.com/jackwener). The core idea — turning major websites into CLI interfaces that reuse existing browser sessions — makes AI agents dramatically more capable without requiring any platform API access.
