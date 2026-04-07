```markdown
---
name: graphify-knowledge-graph
description: Build queryable knowledge graphs from code, docs, papers, and images using AI coding assistant skills
triggers:
  - "graphify my codebase"
  - "build a knowledge graph"
  - "turn my files into a graph"
  - "understand this codebase with graphify"
  - "run graphify on this folder"
  - "query the knowledge graph"
  - "install graphify skill"
  - "extract relationships from my code"
---

# graphify-knowledge-graph

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

graphify turns any folder of code, docs, papers, or images into a queryable knowledge graph. It runs as an AI coding assistant skill — type `/graphify` in Claude Code, Codex, OpenCode, or OpenClaw to extract structure, relationships, and design rationale from your files into an interactive graph you can navigate and query without re-reading raw files.

---

## Install

```bash
pip install graphifyy && graphify install
```

> The PyPI package is `graphifyy`; the CLI and skill command remain `graphify`.

### Platform-specific install

```bash
graphify install                        # Claude Code (default)
graphify install --platform codex       # Codex
graphify install --platform opencode    # OpenCode
graphify install --platform claw        # OpenClaw
```

### Always-on assistant integration (recommended)

Run once per project so your assistant consults the graph before searching files:

```bash
graphify claude install      # writes CLAUDE.md section + PreToolUse hook (Claude Code)
graphify codex install       # writes AGENTS.md (Codex)
graphify opencode install    # writes AGENTS.md (OpenCode)
graphify claw install        # writes AGENTS.md (OpenClaw)
```

Undo with the matching uninstall command:

```bash
graphify claude uninstall
```

### Manual install (curl, no pip)

```bash
mkdir -p ~/.claude/skills/graphify
curl -fsSL https://raw.githubusercontent.com/safishamsi/graphify/v3/graphify/skill.md \
  > ~/.claude/skills/graphify/SKILL.md
```

Add to `~/.claude/CLAUDE.md`:

```
- **graphify** (`~/.claude/skills/graphify/SKILL.md`) - any input to knowledge graph. Trigger: `/graphify`
When the user types `/graphify`, invoke the Skill tool with `skill: "graphify"` before doing anything else.
```

---

## Core workflow

### 1. Build the graph

```bash
# In your AI coding assistant
/graphify .                        # current directory
/graphify ./src                    # specific folder
/graphify ./raw --mode deep        # aggressive INFERRED edge extraction
/graphify ./raw --no-viz           # skip HTML, produce report + JSON only
```

### 2. Outputs

```
graphify-out/
├── graph.html       # interactive — click nodes, search, filter by community
├── GRAPH_REPORT.md  # god nodes, surprising connections, suggested questions
├── graph.json       # persistent graph — query later without re-reading files
└── cache/           # SHA256 cache — re-runs only process changed files
```

### 3. Query the graph

```bash
/graphify query "what connects attention to the optimizer?"
/graphify query "what connects attention to the optimizer?" --dfs        # trace a path
/graphify query "what connects attention to the optimizer?" --budget 1500  # cap tokens
/graphify path "DigestAuth" "Response"      # shortest path between two nodes
/graphify explain "SwinTransformer"         # expand a single node
```

---

## Key commands reference

### Building

| Command | What it does |
|---|---|
| `/graphify .` | Build graph from current directory |
| `/graphify ./folder` | Build from a specific folder |
| `/graphify ./folder --mode deep` | More aggressive INFERRED edge extraction |
| `/graphify ./folder --update` | Re-extract only changed files, merge into existing graph |
| `/graphify ./folder --cluster-only` | Rerun clustering without re-extraction |
| `/graphify ./folder --watch` | Auto-sync as files change (code: instant AST; docs: notifies you) |

### Ingesting remote content

```bash
/graphify add https://arxiv.org/abs/1706.03762          # fetch a paper, add to graph
/graphify add https://x.com/karpathy/status/...         # fetch a tweet
/graphify add https://... --author "Andrej Karpathy"    # tag original author
/graphify add https://... --contributor "Your Name"     # tag who added it
```

### Querying

```bash
/graphify query "why does the auth layer depend on redis?"
/graphify query "what implements the retry protocol?" --dfs
/graphify path "Transformer" "AdamW"
/graphify explain "DigestAuth"
```

### Exporting

```bash
/graphify ./folder --svg              # export graph.svg
/graphify ./folder --graphml          # export graph.graphml (Gephi, yEd)
/graphify ./folder --neo4j            # generate cypher.txt for Neo4j import
/graphify ./folder --neo4j-push bolt://localhost:7687   # push to live Neo4j
/graphify ./folder --obsidian         # generate Obsidian vault (opt-in)
/graphify ./folder --wiki             # build agent-crawlable wiki (index.md + per-community articles)
/graphify ./folder --mcp              # start MCP stdio server
```

### Git hooks

```bash
graphify hook install     # post-commit + post-checkout: auto-rebuild on commit/branch switch
graphify hook uninstall
graphify hook status
```

---

## Supported file types

| Type | Extensions | Extraction method |
|---|---|---|
| Code | `.py .ts .js .go .rs .java .c .cpp .rb .cs .kt .scala .php .swift .lua` | AST via tree-sitter + call graph + docstring/comment rationale (no LLM) |
| Docs | `.md .txt .rst` | Concepts + relationships + design rationale via Claude |
| Papers | `.pdf` | Citation mining + concept extraction |
| Images | `.png .jpg .webp .gif` | Claude vision — screenshots, diagrams, any language |

---

## How it works

graphify runs in **two passes**:

1. **AST pass (deterministic, no LLM):** Extracts classes, functions, imports, call graphs, docstrings, and rationale comments from code files.
2. **Semantic pass (parallel Claude subagents):** Extracts concepts, relationships, and design rationale from docs, papers, and images.

Results are merged into a **NetworkX graph**, clustered with **Leiden community detection** (topology-based — no embeddings or vector database), and exported as HTML, JSON, and a plain-language audit report.

### Edge provenance tags

Every relationship is tagged so you always know what was found vs guessed:

| Tag | Meaning | Confidence |
|---|---|---|
| `EXTRACTED` | Found directly in source | Always 1.0 |
| `INFERRED` | Reasonable inference | 0.0–1.0 score |
| `AMBIGUOUS` | Flagged for human review | — |

---

## Python API

graphify is primarily a CLI/skill tool, but the graph output (`graph.json`) is standard NetworkX JSON you can load and traverse:

```python
import json
import networkx as nx

# Load the persistent graph
with open("graphify-out/graph.json") as f:
    data = json.load(f)

G = nx.node_link_graph(data)

# Find god nodes (highest degree)
god_nodes = sorted(G.degree(), key=lambda x: x[1], reverse=True)[:10]
for node, degree in god_nodes:
    print(f"{node}: {degree} connections")

# Find all EXTRACTED edges (high confidence, found in source)
extracted_edges = [
    (u, v, d) for u, v, d in G.edges(data=True)
    if d.get("provenance") == "EXTRACTED"
]

# Find INFERRED edges above a confidence threshold
high_confidence_inferred = [
    (u, v, d) for u, v, d in G.edges(data=True)
    if d.get("provenance") == "INFERRED" and d.get("confidence_score", 0) > 0.85
]

# Shortest path between two concepts
try:
    path = nx.shortest_path(G, source="DigestAuth", target="Response")
    print(" -> ".join(path))
except nx.NetworkXNoPath:
    print("No path found")

# Get all nodes in a community
communities = {}
for node, data in G.nodes(data=True):
    community_id = data.get("community")
    if community_id is not None:
        communities.setdefault(community_id, []).append(node)

for cid, members in sorted(communities.items()):
    print(f"Community {cid}: {', '.join(members[:5])}{'...' if len(members) > 5 else ''}")
```

### Working with rationale nodes

graphify extracts `# NOTE:`, `# IMPORTANT:`, `# HACK:`, `# WHY:` comments and docstrings as `rationale_for` nodes:

```python
# Find all rationale nodes and what they explain
rationale_nodes = [
    (node, data) for node, data in G.nodes(data=True)
    if data.get("node_type") == "rationale_for"
]

for node, data in rationale_nodes:
    print(f"Rationale: {data.get('label')}")
    # Find what this rationale is connected to
    neighbors = list(G.neighbors(node))
    print(f"  Explains: {neighbors}")
```

### Querying semantic similarity edges

```python
# Find cross-file semantic links (concepts connected without structural relationship)
semantic_edges = [
    (u, v, d) for u, v, d in G.edges(data=True)
    if d.get("relation") == "semantically_similar_to"
]

for u, v, data in semantic_edges:
    score = data.get("confidence_score", 0)
    print(f"{u} ~ {v} (confidence: {score:.2f})")
```

---

## Common patterns

### Pattern 1: Onboard to an unfamiliar codebase

```bash
# Install graphify, build the graph, read the report
pip install graphifyy && graphify install

# In Claude Code
/graphify .

# Read the output — god nodes tell you what everything routes through
cat graphify-out/GRAPH_REPORT.md
```

### Pattern 2: Mixed research corpus (Karpathy-style `/raw` folder)

Drop code, PDFs, screenshots, and notes in one folder:

```
raw/
├── attention_is_all_you_need.pdf
├── training_notes.md
├── whiteboard_photo.png
├── nanoGPT/
│   └── model.py
└── tweet_screenshot.jpg
```

```bash
/graphify ./raw
```

graphify uses Claude vision on images, citation mining on PDFs, AST on code, and semantic extraction on markdown — all merged into one graph.

### Pattern 3: Incremental updates (large codebase)

```bash
# First full build
/graphify ./src

# After making changes — only re-processes changed files via SHA256 cache
/graphify ./src --update

# After a major refactor — rerun clustering without re-extracting
/graphify ./src --cluster-only
```

### Pattern 4: Live development with auto-sync

```bash
# Terminal 1: keep graph in sync as you code
/graphify ./src --watch

# Terminal 2: your normal development
# Code saves → instant AST rebuild
# Doc/image saves → graphify notifies you to run --update for LLM re-pass
```

### Pattern 5: Export to external tools

```bash
# Neo4j (generate Cypher, then push)
/graphify ./src --neo4j
# cypher.txt is written to graphify-out/
/graphify ./src --neo4j-push bolt://localhost:7687

# Gephi / yEd
/graphify ./src --graphml

# Obsidian vault
/graphify ./src --obsidian

# Agent-crawlable wiki
/graphify ./src --wiki
# graphify-out/wiki/index.md is the entry point
```

### Pattern 6: Git hook integration

```bash
# Auto-rebuild graph on every commit and branch switch
graphify hook install

# Verify hooks are active
graphify hook status
```

### Pattern 7: MCP server

```bash
# Start an MCP stdio server so any MCP-compatible client can query the graph
/graphify ./src --mcp
```

---

## Understanding GRAPH_REPORT.md

The report has four sections:

```markdown
## God Nodes
Highest-degree concepts — what everything connects through.
These are your architectural load-bearing walls.

## Surprising Connections
Cross-domain edges ranked by composite score.
Code-paper edges rank higher than code-code.
Each result includes a plain-English why.

## Suggested Questions
4-5 questions the graph is uniquely positioned to answer.
Start here when exploring an unfamiliar corpus.

## Token Benchmark
Printed after every run.
First run: extracts and builds (costs tokens).
Subsequent queries: read compact graph.json instead of raw files.
SHA256 cache means re-runs only re-process changed files.
```

---

## Troubleshooting

### `graphify: command not found` after pip install

```bash
# Check your PATH includes pip's script directory
python -m graphify install

# Or use the full path
python -m pip show graphifyy | grep Location
# then add {Location}/../Scripts to PATH (Windows) or {Location}/../bin (Unix)
```

### Skill not triggering in Claude Code

Verify install wrote the skill file:

```bash
ls ~/.claude/skills/graphify/SKILL.md
```

Check `~/.claude/CLAUDE.md` contains the graphify entry. If missing, re-run:

```bash
graphify install
```

### LLM extraction fails / no semantic edges

Parallel subagents require the AI assistant to support multi-agent mode:

- **Codex:** add `multi_agent = true` under `[features]` in `~/.codex/config.toml`
- **OpenClaw:** uses sequential extraction (parallel agent support is early-stage)
- **Claude Code / OpenCode:** parallel extraction works by default

### Graph has no god nodes or surprising connections

The corpus may be too small (< ~6 files). At small scale, graphify still works but token reduction is minimal — the value is structural clarity, not compression. Try `--mode deep` for more aggressive INFERRED edge extraction:

```bash
/graphify ./src --mode deep
```

### Re-run processes all files instead of only changed files

The SHA256 cache lives in `graphify-out/cache/`. If it's missing or the output folder was deleted, graphify re-processes everything. This is expected. Subsequent runs use the cache.

```bash
ls graphify-out/cache/    # verify cache exists after first run
```

### Neo4j push fails

```bash
# Verify Neo4j is running and bolt port is accessible
/graphify ./src --neo4j                          # generate cypher.txt first
# then manually import in Neo4j Browser:
# :source graphify-out/cypher.txt

# Or push directly (requires neo4j Python driver)
pip install neo4j
/graphify ./src --neo4j-push bolt://localhost:7687
```

Set credentials via environment variables — do not hardcode:

```bash
export NEO4J_USERNAME=neo4j
export NEO4J_PASSWORD=your_password
```

### `--watch` misses doc/image changes

Watch mode gives instant rebuilds for **code** files (AST only, no LLM). For docs and images, it prints a notification because LLM re-extraction is non-trivial to run on every save. When you see the notification:

```bash
/graphify ./src --update    # re-processes only changed docs/images
```

---

## Token efficiency

graphify prints a benchmark after every run. Typical results:

| Corpus | Files | Token reduction |
|---|---|---|
| Karpathy repos + 5 papers + 4 images | 52 | **71.5x** |
| graphify source + Transformer paper | 4 | **5.4x** |
| Small Python library | 6 | ~1x |

Token reduction scales with corpus size. At 52 mixed files, querying `graph.json` uses 71x fewer tokens than reading raw files. The first run costs tokens to build the graph. Every subsequent query reads the compact graph instead — savings compound across sessions.
```
