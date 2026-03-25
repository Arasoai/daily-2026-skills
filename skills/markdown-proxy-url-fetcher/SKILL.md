```markdown
---
name: markdown-proxy-url-fetcher
description: Fetch any URL as clean Markdown via proxy cascade (r.jina.ai / defuddle.md / agent-fetch) with built-in support for WeChat, Feishu/Lark docs, and login-required pages
triggers:
  - fetch this URL as markdown
  - convert webpage to markdown
  - read this article for me
  - scrape this WeChat article
  - fetch this Feishu doc
  - get content from this URL
  - extract markdown from webpage
  - read this tweet or X post
---

# markdown-proxy URL Fetcher

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Convert any URL to clean Markdown with automatic fallback across multiple proxy services. Handles login-required pages including X/Twitter, WeChat public accounts, and Feishu/Lark documents.

## What It Does

| URL Type | Method | Notes |
|----------|--------|-------|
| WeChat (`mp.weixin.qq.com`) | Playwright headless browser | Bypasses anti-scraping |
| Feishu/Lark docs (`feishu.cn`, `larksuite.com`) | Feishu Open API | Requires app credentials |
| YouTube | Dedicated YouTube skill | Not handled by this skill |
| All other URLs | Proxy cascade | Free, no API key needed |

### Proxy Cascade Order
1. **r.jina.ai** — Most complete, preserves images
2. **defuddle.md** — Cleaner output with YAML frontmatter
3. **agent-fetch** — Local fallback, no network proxy needed
4. **defuddle CLI** — Local CLI fallback

## Installation

```bash
npx skills add joeseesun/markdown-proxy
```

Verify installation:
```bash
ls ~/.claude/skills/markdown-proxy/SKILL.md
```

## Prerequisites

### Core (always needed)
- `curl` — built-in on macOS/Linux

### WeChat scraping
```bash
pip install playwright beautifulsoup4 lxml
playwright install chromium
```

### Feishu/Lark docs
```bash
export FEISHU_APP_ID=your_app_id
export FEISHU_APP_SECRET=your_app_secret
```

### Proxy fallback
```bash
npx agent-fetch --help  # npx auto-downloads, no pre-install needed
```

## Usage with Claude Code

Just give Claude a URL in natural language:

```
Read this article: https://example.com/post
Fetch this tweet: https://x.com/user/status/123456
Read this WeChat article: https://mp.weixin.qq.com/s/abc123
Convert this Feishu doc: https://company.feishu.cn/docx/AbCdEfGh
Read this Feishu wiki page: https://company.feishu.cn/wiki/AbCdEfGh
```

## How the Proxy Cascade Works

```bash
# Step 1: Try r.jina.ai
curl -s "https://r.jina.ai/https://example.com/article"

# Step 2: If empty/failed, try defuddle.md
curl -s "https://defuddle.md/https://example.com/article"

# Step 3: If still failed, try agent-fetch locally
npx agent-fetch https://example.com/article

# Step 4: Last resort — defuddle CLI
npx defuddle https://example.com/article
```

## WeChat Scraping (Built-in Script)

The skill includes `fetch_wechat.py` using Playwright:

```python
# The script is bundled — Claude Code invokes it automatically
# Manual usage:
python ~/.claude/skills/markdown-proxy/fetch_wechat.py \
  "https://mp.weixin.qq.com/s/your_article_id"
```

What the script does internally:
```python
from playwright.sync_api import sync_playwright
from bs4 import BeautifulSoup

def fetch_wechat_article(url: str) -> str:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        page.goto(url, wait_until="networkidle")
        html = page.content()
        browser.close()

    soup = BeautifulSoup(html, "lxml")
    # Extracts #js_content div (WeChat article body)
    article = soup.find(id="js_content")
    return convert_to_markdown(article)
```

## Feishu/Lark Document Support

The bundled `fetch_feishu.py` script uses Feishu Open API:

```bash
# Set credentials (required)
export FEISHU_APP_ID=cli_xxxxxxxxxxxxxxxx
export FEISHU_APP_SECRET=your_secret_here

# Manual invocation
python ~/.claude/skills/markdown-proxy/fetch_feishu.py \
  "https://company.feishu.cn/docx/AbCdEfGhIjKl"
```

### Supported Feishu URL Formats
```
https://{tenant}.feishu.cn/docx/{doc_id}     # New docs
https://{tenant}.feishu.cn/docs/{doc_id}     # Legacy docs
https://{tenant}.feishu.cn/wiki/{wiki_id}    # Wiki pages
https://{tenant}.larksuite.com/docx/{doc_id} # Lark (international)
```

### Required Feishu App Permissions
- `docx:document:readonly` — for docx and doc files
- `wiki:wiki:readonly` — for wiki pages

### How Feishu API Fetching Works
```python
import os, requests

def get_feishu_token():
    resp = requests.post(
        "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
        json={
            "app_id": os.environ["FEISHU_APP_ID"],
            "app_secret": os.environ["FEISHU_APP_SECRET"],
        }
    )
    return resp.json()["tenant_access_token"]

def fetch_doc_blocks(doc_id: str, token: str):
    headers = {"Authorization": f"Bearer {token}"}
    resp = requests.get(
        f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks",
        headers=headers
    )
    return resp.json()["data"]["items"]
```

### Supported Feishu Block Types → Markdown
| Block Type | Markdown Output |
|------------|-----------------|
| Heading 1-9 | `# ` to `######### ` |
| Bullet list | `- item` |
| Ordered list | `1. item` |
| Code block | ` ```lang\ncode\n``` ` |
| Quote | `> text` |
| Todo | `- [ ] task` / `- [x] done` |
| Equation | `$$math$$` |
| Image | `![alt](url)` |

## Real Code Examples

### Fetch Any URL Programmatically
```python
import subprocess

def fetch_as_markdown(url: str) -> str:
    """Try proxy cascade to get URL content as Markdown."""
    # Try r.jina.ai first
    result = subprocess.run(
        ["curl", "-s", f"https://r.jina.ai/{url}"],
        capture_output=True, text=True, timeout=30
    )
    if result.stdout.strip():
        return result.stdout

    # Fall back to defuddle.md
    result = subprocess.run(
        ["curl", "-s", f"https://defuddle.md/{url}"],
        capture_output=True, text=True, timeout=30
    )
    if result.stdout.strip():
        return result.stdout

    # Fall back to agent-fetch
    result = subprocess.run(
        ["npx", "agent-fetch", url],
        capture_output=True, text=True, timeout=60
    )
    return result.stdout or "Failed to fetch content"
```

### Route by URL Type
```python
def smart_fetch(url: str) -> str:
    if "mp.weixin.qq.com" in url:
        return subprocess.run(
            ["python", "fetch_wechat.py", url],
            capture_output=True, text=True
        ).stdout
    elif any(d in url for d in ["feishu.cn", "larksuite.com"]):
        return subprocess.run(
            ["python", "fetch_feishu.py", url],
            capture_output=True, text=True
        ).stdout
    else:
        return fetch_as_markdown(url)
```

### Using r.jina.ai with Custom Headers
```bash
# Basic fetch
curl -s "https://r.jina.ai/https://example.com/article"

# With X-Return-Format header for specific output
curl -s \
  -H "X-Return-Format: markdown" \
  "https://r.jina.ai/https://example.com/article"

# With timeout
curl -s --max-time 20 "https://r.jina.ai/https://example.com/article"
```

### Using defuddle.md
```bash
# Returns Markdown with YAML frontmatter (title, author, date)
curl -s "https://defuddle.md/https://example.com/article"

# Output format:
# ---
# title: Article Title
# author: Author Name
# date: 2026-03-21
# ---
# # Article Title
# Content...
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| WeChat fetch fails | `playwright install chromium` |
| Feishu returns 403 | Verify `FEISHU_APP_ID` + `FEISHU_APP_SECRET` are set; check app permissions in Feishu admin |
| Feishu wiki fails | Add `wiki:wiki:readonly` permission to your Feishu app |
| r.jina.ai returns empty | Automatic — falls back to defuddle.md |
| All proxies fail | URL has strict auth; try `npx agent-fetch <url>` manually |
| Playwright not found | `pip install playwright && playwright install chromium` |
| agent-fetch slow | First run downloads via npx; subsequent runs are cached |

### Debug Proxy Chain Manually
```bash
# Test each step
echo "=== r.jina.ai ===" && curl -s "https://r.jina.ai/https://example.com" | head -20
echo "=== defuddle.md ===" && curl -s "https://defuddle.md/https://example.com" | head -20
echo "=== agent-fetch ===" && npx agent-fetch https://example.com 2>&1 | head -20
```

### Verify Feishu Credentials
```bash
# Test token generation
curl -X POST "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\": \"$FEISHU_APP_ID\", \"app_secret\": \"$FEISHU_APP_SECRET\"}"
# Should return: {"code":0,"msg":"ok","tenant_access_token":"...","expire":7200}
```

## Common Patterns

### Batch Fetch Multiple URLs
```python
urls = [
    "https://example.com/post-1",
    "https://example.com/post-2",
    "https://mp.weixin.qq.com/s/abc123",
]

results = {}
for url in urls:
    results[url] = smart_fetch(url)
    print(f"Fetched: {url} ({len(results[url])} chars)")
```

### Save Fetched Markdown to File
```bash
curl -s "https://r.jina.ai/https://example.com/article" > article.md
# Or with defuddle for cleaner frontmatter:
curl -s "https://defuddle.md/https://example.com/article" > article.md
```

### Integration with Claude Code Workflow
```
# In Claude Code conversation:
User: "Summarize this paper: https://arxiv.org/abs/2401.12345"
# Claude will:
# 1. Detect it's a standard URL
# 2. Use r.jina.ai proxy to fetch content
# 3. Fall back through cascade if needed
# 4. Return summary based on fetched Markdown
```
```
