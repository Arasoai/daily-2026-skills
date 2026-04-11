```markdown
---
name: claude-obsidian-wiki
description: Claude Code skill for building and maintaining a persistent, compounding Obsidian wiki vault using AI-powered ingestion, querying, linting, and autonomous research.
triggers:
  - "set up claude obsidian wiki"
  - "build a second brain with obsidian"
  - "ingest sources into my wiki vault"
  - "query my obsidian knowledge base"
  - "lint my obsidian wiki"
  - "run autoresearch on a topic"
  - "save this conversation to my wiki"
  - "set up persistent knowledge base with claude"
---

# claude-obsidian-wiki

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

claude-obsidian is a Claude Code skill that turns Obsidian into a self-organizing AI knowledge engine. Drop sources, ask questions, run autonomous research — Claude handles ingestion, cross-referencing, contradiction flagging, session memory, and vault health. Knowledge compounds with every ingest.

Based on [Andrej Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). 10 skills. Zero manual filing. Multi-agent support.

---

## Installation

### Option 1: Clone as vault (recommended)

```bash
git clone https://github.com/AgriciDaniel/claude-obsidian
cd claude-obsidian
bash bin/setup-vault.sh
```

Open the folder in Obsidian: **Manage Vaults → Open folder as vault → select `claude-obsidian/`**

Open Claude Code in the same folder. Type `/wiki` to begin.

`setup-vault.sh` configures `graph.json`, `app.json`, and `appearance.json`. Run it once before first Obsidian open.

### Option 2: Claude Code plugin install

```bash
# Add the marketplace catalog
claude plugin marketplace add AgriciDaniel/claude-obsidian

# Install the plugin
claude plugin install claude-obsidian@claude-obsidian-marketplace

# Verify
claude plugin list
```

Then in any Claude Code session: `/wiki`

### Option 3: Add to an existing vault

```bash
# Copy WIKI.md into your vault root, then paste into Claude:
```

```
Read WIKI.md in this project. Then:
1. Check if Obsidian is installed. If not, install it.
2. Check if the Local REST API plugin is running on port 27124.
3. Configure the MCP server.
4. Ask me ONE question: "What is this vault for?"
Then scaffold the full wiki structure.
```

---

## MCP Setup (Optional but Recommended)

MCP lets Claude read and write vault notes directly without copy-paste.

### Option A: REST API (requires Obsidian Local REST API plugin)

1. Install **Local REST API** plugin in Obsidian
2. Copy your API key from plugin settings
3. Run:

```bash
claude mcp add-json obsidian-vault '{
  "type": "stdio",
  "command": "uvx",
  "args": ["mcp-obsidian"],
  "env": {
    "OBSIDIAN_API_KEY": "$OBSIDIAN_API_KEY",
    "OBSIDIAN_HOST": "127.0.0.1",
    "OBSIDIAN_PORT": "27124",
    "NODE_TLS_REJECT_UNAUTHORIZED": "0"
  }
}' --scope user
```

Set the env var before running:
```bash
export OBSIDIAN_API_KEY=your-key-from-plugin-settings
```

### Option B: Filesystem (no plugin needed)

```bash
claude mcp add-json obsidian-vault '{
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@bitbonsai/mcpvault@latest", "/path/to/your/vault"]
}' --scope user
```

---

## Core Commands

| Command | What Claude does |
|---|---|
| `/wiki` | Setup check, scaffold structure, or resume last session |
| `ingest [file or URL]` | Read source, create 8–15 wiki pages, update index and log |
| `ingest all of these` | Batch process multiple sources, then cross-reference |
| `what do you know about X?` | Read index → relevant pages → synthesize answer with citations |
| `/save` | File current conversation as a wiki note (prompts for name) |
| `/save [name]` | Save with a specific title, skips naming question |
| `/autoresearch [topic]` | 3-round autonomous research: search, fetch, synthesize, file |
| `/canvas` | Open or create visual canvas, list zones and nodes |
| `lint the wiki` | Health check: orphans, dead links, gaps, missing cross-refs |
| `update hot cache` | Refresh `wiki/hot.md` with latest context summary |

---

## Vault Structure

After `/wiki` scaffolds your vault:

```
your-vault/
├── wiki/
│   ├── index.md          # Master catalog of all wiki pages
│   ├── log.md            # Append-only operation log
│   ├── hot.md            # Recent context cache (session memory)
│   ├── overview.md       # Executive summary
│   └── meta/
│       ├── dashboard.base    # Native Obsidian Bases dashboard
│       └── dashboard.md      # Legacy Dataview dashboard (fallback)
├── _templates/           # Templater templates for each note type
├── .obsidian/
│   └── snippets/
│       └── vault-colors.css  # Color-coded file explorer
└── CLAUDE.md             # Auto-loaded project instructions
```

---

## Wiki Modes

Select one or combine during `/wiki` setup:

| Mode | Use case |
|---|---|
| A: Website | Sitemap, content audit, SEO wiki |
| B: GitHub | Codebase map, architecture wiki |
| C: Business | Project wiki, competitive intelligence |
| D: Personal | Second brain, goals, journal synthesis |
| E: Research | Papers, concepts, thesis |
| F: Book/Course | Chapter tracker, course notes |

---

## Ingestion Workflow

### Single source

```
ingest research-paper.pdf
```

Claude will:
1. Read the source
2. Extract entities, concepts, and key claims
3. Create 8–15 wiki pages with proper frontmatter
4. Update `wiki/index.md` with new entries
5. Append to `wiki/log.md`
6. Cross-reference related existing pages
7. Flag contradictions with `[!contradiction]` callouts

### Batch ingestion

```
ingest all of these:
- paper1.pdf
- paper2.pdf
- https://example.com/article
- notes.md
```

Claude spawns parallel agents per source, then runs a cross-reference pass.

### Web Clipper workflow

Install the [Obsidian Web Clipper](https://obsidian.md/clipper) browser extension. Pages saved via clipper land in `.raw/`. Then:

```
ingest everything in .raw/
```

---

## Query Workflow

```
what do you know about transformer attention mechanisms?
```

Claude's resolution order:
1. Reads `wiki/hot.md` (recent context cache)
2. Scans `wiki/index.md` for relevant pages
3. Drills into matching wiki pages
4. Synthesizes answer with citations to specific wiki pages — not training data

---

## Autonomous Research

```
/autoresearch "retrieval augmented generation"
```

Claude runs 3 rounds:
1. Web search for the topic
2. Fetch and read top sources
3. Synthesize findings, identify gaps, repeat
4. File all results as wiki pages with cross-references

Configure research objectives in:
```
skills/autoresearch/references/program.md
```

---

## Session Memory (Hot Cache)

`wiki/hot.md` persists recent context between sessions. At session end:

```
update hot cache
```

At session start, Claude automatically reads `hot.md` — no recap needed.

---

## Lint the Wiki

```
lint the wiki
```

Claude checks 8 categories:
- **Orphans**: pages with no incoming links
- **Dead links**: internal links pointing to missing pages
- **Stale claims**: content that may be outdated
- **Missing cross-references**: related pages not yet linked
- **Duplicate concepts**: similar pages that could be merged
- **Missing frontmatter**: pages lacking required properties
- **Empty sections**: placeholder headings with no content
- **Index gaps**: pages not cataloged in `wiki/index.md`

---

## Cross-Project Knowledge Base

Point any Claude Code project at this vault. Add to that project's `CLAUDE.md`:

```markdown
## Wiki Knowledge Base
Path: ~/path/to/claude-obsidian

When you need context not already in this project:
1. Read wiki/hot.md first (recent context cache)
2. If not enough, read wiki/index.md
3. If you need domain details, read the relevant domain sub-index
4. Only then drill into specific wiki pages

Do NOT read the wiki for general coding questions unrelated to [your domain].
```

---

## Canvas Commands

```bash
/canvas                          # Open or create the visual canvas
/canvas add image [path-or-url]  # Add image with auto-layout
/canvas add text [content]       # Add markdown text card
/canvas add pdf [path]           # Add PDF as rendered preview node
/canvas add note [wiki-page]     # Pin a wiki page as linked card
/canvas zone [name]              # Add labeled zone for organization
/canvas from banana              # Capture recently generated images
```

Pairs with [claude-canvas](https://github.com/AgriciDaniel/claude-canvas) for 12 templates, 6 layout algorithms, and AI image generation.

---

## Wiki Note Frontmatter

All ingested pages use this frontmatter schema:

```yaml
---
title: "Attention Is All You Need"
type: source          # source | concept | entity | hub | log
tags: [transformers, attention, nlp]
created: 2026-04-11
updated: 2026-04-11
status: current       # current | stale | stub
source: "https://arxiv.org/abs/1706.03762"
related:
  - "[[Transformer Architecture]]"
  - "[[Self-Attention Mechanism]]"
banner: "_attachments/images/transformer.png"
banner_icon: "🤖"
---
```

---

## Contradiction Callout Pattern

When Claude detects conflicting information across sources:

```markdown
> [!contradiction]
> **Source A** (2024) claims X. **Source B** (2025) claims Y.
> Last verified: 2026-04-11. Needs resolution.
```

---

## Obsidian Plugins

### Pre-installed (ship with vault)

| Plugin | Purpose |
|---|---|
| Calendar | Right-sidebar calendar with word count + task dots |
| Thino | Quick memo capture panel |
| Excalidraw | Freehand drawing (downloaded by `setup-vault.sh`) |
| Banners | Notion-style header images via `banner:` frontmatter |

### Install from Community Plugins

| Plugin | Purpose |
|---|---|
| Templater | Auto-fills frontmatter from `_templates/` |
| Obsidian Git | Auto-commits vault every 15 minutes |
| Local REST API | Required for MCP Option A |

### Core Plugins (built-in, no install)

- **Bases** — powers `dashboard.base` (requires Obsidian v1.9.10+)
- **Properties** — visual frontmatter editor
- **Backlinks**, **Graph view**, **Outline**

---

## CSS Snippets (auto-enabled)

| Snippet | Effect |
|---|---|
| `vault-colors` | Color-codes `wiki/` folders: blue=concepts, green=sources, purple=entities |
| `ITS-Dataview-Cards` | Turns Dataview TABLE queries into visual card grids |
| `ITS-Image-Adjustments` | Fine-grained image sizing: append `\|100` to any image embed |

---

## Troubleshooting

**`/wiki` doesn't respond**
```bash
# Confirm plugin is installed
claude plugin list

# Reinstall if missing
claude plugin install claude-obsidian@claude-obsidian-marketplace
```

**MCP connection refused**
```bash
# Check Local REST API plugin is running in Obsidian
# Verify port
curl http://127.0.0.1:27124/vault/ -H "Authorization: Bearer $OBSIDIAN_API_KEY"

# Check env var is set
echo $OBSIDIAN_API_KEY
```

**`setup-vault.sh` fails on Excalidraw download**
```bash
# Run manually
bash bin/setup-vault.sh

# Or skip Excalidraw — enable it later from Community Plugins
```

**Graph view looks empty**
```bash
# Re-run setup to apply graph.json filter + colors
bash bin/setup-vault.sh

# Then restart Obsidian
```

**Hot cache is stale**
```
update hot cache
```
Run this at the end of every significant session.

**Dataview dashboard shows errors**
The primary dashboard now uses Bases (Obsidian v1.9.10+). The legacy `dashboard.md` requires the Dataview community plugin. Install it if you need the fallback view.

---

## File Structure Reference

```
claude-obsidian/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Distribution metadata
├── skills/
│   ├── wiki/                # Orchestrator + 7 reference files
│   ├── wiki-ingest/         # INGEST operation skill
│   ├── wiki-query/          # QUERY operation skill
│   ├── wiki-lint/           # LINT operation skill
│   ├── save/                # /save skill
│   ├── autoresearch/        # /autoresearch skill
│   │   └── references/
│   │       └── program.md   # Configurable research objectives
│   └── canvas/              # /canvas visual layer skill
│       └── references/
├── bin/
│   └── setup-vault.sh       # One-time vault configuration
├── wiki/                    # Your actual knowledge base
│   ├── index.md
│   ├── log.md
│   ├── hot.md
│   └── meta/
└── WIKI.md                  # Standalone install instructions
```
```
