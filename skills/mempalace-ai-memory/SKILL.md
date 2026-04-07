```markdown
---
name: mempalace-ai-memory
description: MemPalace is a local-first AI memory system using palace-structure retrieval and AAAK compression, achieving 96.6%+ recall on LongMemEval benchmarks.
triggers:
  - set up AI memory for my project
  - remember my conversations with Claude
  - search my past AI chat history
  - connect memory to my local LLM
  - store and retrieve AI conversation context
  - set up MCP memory server
  - mine my chat exports for memory
  - query what we decided about a topic
---

# MemPalace AI Memory System

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

MemPalace is a local-first AI memory system that stores all your AI conversations and makes them searchable with 96.6%+ recall. It organizes memory into a hierarchical "palace" structure (wings → rooms → closets → drawers) and uses AAAK — a lossless compression dialect — to deliver months of context in ~170 tokens. Runs entirely on your machine with no cloud API required.

---

## Installation

```bash
pip install mempalace
```

Verify install:

```bash
mempalace --version
mempalace status
```

---

## Initial Setup

### 1. Initialize a Palace

```bash
# Initialize for a specific project directory
mempalace init ~/projects/myapp

# This creates ~/.mempalace/palace/ with the wing/room/closet/drawer structure
# MemPalace auto-detects rooms (topics) from the project
```

### 2. Mine Your Data

Three mining modes:

```bash
# Mode: projects — code, docs, notes
mempalace mine ~/projects/myapp

# Mode: convos — Claude, ChatGPT, Slack exports
mempalace mine ~/chats/ --mode convos

# Mode: general — auto-classifies into decisions, preferences,
#                 milestones, problems, and emotional context
mempalace mine ~/chats/ --mode convos --extract general
```

Mining is a one-time operation per dataset. Re-run when you have new exports.

### 3. Verify

```bash
mempalace status
# Shows wings, rooms, and memory counts
```

---

## Key CLI Commands

| Command | Purpose |
|---|---|
| `mempalace init <path>` | Initialize palace for a project |
| `mempalace mine <path>` | Mine project files into memory |
| `mempalace mine <path> --mode convos` | Mine conversation exports |
| `mempalace mine <path> --mode convos --extract general` | Mine + classify |
| `mempalace search "<query>"` | Search all memories |
| `mempalace wake-up` | Output critical context (~170 tokens) for LLM injection |
| `mempalace status` | Show palace structure and memory stats |

---

## MCP Server (Claude, ChatGPT, Cursor)

Connect once and your AI uses memory automatically:

```bash
# Add MemPalace as an MCP server in Claude Code
claude mcp add mempalace -- python -m mempalace.mcp_server
```

After connecting, Claude gets 19 tools including `mempalace_search`. Users just ask naturally:

> *"What did we decide about auth last month?"*

Claude calls `mempalace_search` automatically — no manual commands needed.

### Available MCP Tools (subset)

- `mempalace_search` — semantic search across all memories
- `mempalace_wake_up` — load critical facts into context
- `mempalace_add_memory` — store a new memory from conversation
- `mempalace_list_wings` — list all people/projects in the palace
- `mempalace_list_rooms` — list rooms within a wing

---

## Python API

### Search Memories

```python
from mempalace.searcher import search_memories

results = search_memories(
    query="why did we switch to GraphQL",
    palace_path="~/.mempalace/palace",
    top_k=5
)

for result in results:
    print(result["content"])
    print(result["source"])   # drawer path
    print(result["score"])    # relevance score
```

### Store a Memory

```python
from mempalace.memory import store_memory

store_memory(
    content="Decided to use PostgreSQL over MySQL for JSONB support.",
    wing="myapp-project",
    room="decisions",
    palace_path="~/.mempalace/palace"
)
```

### Load Wake-Up Context

```python
from mempalace.wakeup import generate_wakeup

context = generate_wakeup(
    palace_path="~/.mempalace/palace",
    use_aaak=True   # AAAK compression — ~170 tokens
)

# Inject into your local LLM's system prompt
system_prompt = f"You have access to the following memory context:\n{context}"
```

### Mine Conversations Programmatically

```python
from mempalace.miner import mine_directory

mine_directory(
    source_path="~/chats/",
    mode="convos",
    extract="general",          # decisions, preferences, milestones, problems
    palace_path="~/.mempalace/palace"
)
```

---

## Local LLM Integration (Offline)

### Option 1: Wake-Up Injection

```bash
mempalace wake-up > context.txt
```

```python
import subprocess

context = subprocess.check_output(["mempalace", "wake-up"]).decode()

# Inject into Ollama / llama.cpp / any local LLM
import ollama

response = ollama.chat(
    model="llama3",
    messages=[
        {"role": "system", "content": f"Memory context:\n{context}"},
        {"role": "user",   "content": "What did we decide about the database?"}
    ]
)
```

### Option 2: On-Demand Search → Prompt Injection

```python
from mempalace.searcher import search_memories
import ollama

query = "auth implementation decisions"
memories = search_memories(query, palace_path="~/.mempalace/palace", top_k=5)
memory_text = "\n\n".join(m["content"] for m in memories)

response = ollama.chat(
    model="mistral",
    messages=[
        {
            "role": "system",
            "content": f"Relevant memory context:\n{memory_text}"
        },
        {"role": "user", "content": query}
    ]
)
print(response["message"]["content"])
```

---

## Palace Structure

```
~/.mempalace/palace/
├── wings/
│   ├── myapp-project/
│   │   ├── rooms/
│   │   │   ├── decisions/
│   │   │   │   ├── closet/     ← compressed summaries (fast AI reads)
│   │   │   │   └── drawers/    ← original verbatim files (never lost)
│   │   │   ├── auth/
│   │   │   └── billing/
│   │   └── halls.json          ← connections between rooms
│   └── john-doe/
│       └── rooms/
│           └── preferences/
├── tunnels.json                ← cross-wing connections
└── palace.json                 ← palace metadata
```

- **Wings** — a person or project
- **Rooms** — topics within a wing (auto-detected or custom)
- **Halls** — links between related rooms in the same wing
- **Tunnels** — links between rooms across different wings
- **Closets** — compressed summaries pointing to drawers (fast retrieval)
- **Drawers** — verbatim original content (never summarized or lost)

---

## AAAK Compression

AAAK is MemPalace's internal compression dialect — structured text readable by any LLM, no decoder needed.

```python
from mempalace.aaak import encode_to_aaak, decode_from_aaak

# Encode a large context block
aaak_text = encode_to_aaak(long_context_string)
# ~30x compression, zero information loss

# Decode back for human reading
original = decode_from_aaak(aaak_text)
```

AAAK is injected automatically in `wake-up` output when `--aaak` flag is used:

```bash
mempalace wake-up --aaak
# Returns ~170 tokens covering your full palace summary
```

---

## Common Patterns

### Pattern: Daily Conversation Mining

```python
import schedule
from mempalace.miner import mine_directory

def nightly_mine():
    mine_directory(
        source_path="~/Downloads/claude-exports/",
        mode="convos",
        extract="general",
        palace_path="~/.mempalace/palace"
    )

schedule.every().day.at("02:00").do(nightly_mine)
```

### Pattern: Search Before Answering

```python
from mempalace.searcher import search_memories

def answer_with_memory(user_question: str, llm_client) -> str:
    # Retrieve relevant memories first
    memories = search_memories(
        query=user_question,
        palace_path="~/.mempalace/palace",
        top_k=5
    )

    context_blocks = [m["content"] for m in memories if m["score"] > 0.7]
    context = "\n---\n".join(context_blocks)

    return llm_client.complete(
        system=f"Past context:\n{context}" if context else "",
        user=user_question
    )
```

### Pattern: Haiku Reranking (100% Recall Mode)

```python
import anthropic
from mempalace.searcher import search_memories

client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var

def search_with_rerank(query: str, top_k: int = 5):
    # Fetch wider candidate set
    candidates = search_memories(query, top_k=20)

    # Rerank with Claude Haiku (cheap, fast)
    ranked = client.messages.create(
        model="claude-haiku-20240307",
        max_tokens=512,
        system="Rerank these memory chunks by relevance to the query. Return indices only.",
        messages=[{
            "role": "user",
            "content": f"Query: {query}\n\nChunks:\n" +
                       "\n".join(f"[{i}] {c['content'][:200]}"
                                 for i, c in enumerate(candidates))
        }]
    )

    # Parse and return top_k
    return candidates[:top_k]  # apply parsed order
```

---

## Supported Chat Export Formats

MemPalace's `--mode convos` parser handles:

- Claude conversation exports (JSON)
- ChatGPT conversation exports (JSON)
- Slack workspace exports
- Plain text transcripts

```bash
mempalace mine ~/Downloads/conversations.json --mode convos
mempalace mine ~/slack-export/ --mode convos --extract general
```

---

## Troubleshooting

### "No memories found" after mining

```bash
# Check palace was initialized
mempalace status

# Re-run mining with verbose output
mempalace mine ~/chats/ --mode convos --verbose

# Confirm ChromaDB is populated
python -c "
import chromadb
client = chromadb.PersistentClient(path='~/.mempalace/palace/chroma')
print(client.list_collections())
"
```

### MCP server not connecting

```bash
# Test the server directly
python -m mempalace.mcp_server --test

# Re-add with explicit python path
which python  # copy this path
claude mcp add mempalace -- /full/path/to/python -m mempalace.mcp_server
```

### Low recall on searches

```bash
# Use broader queries — semantic search, not keyword
mempalace search "database choice reasoning"   # good
mempalace search "postgres"                    # too narrow

# Enable Haiku reranking for 100% recall
mempalace search "database choice" --rerank    # uses ANTHROPIC_API_KEY
```

### Wake-up output is too large

```bash
# Use AAAK compression (~30x reduction)
mempalace wake-up --aaak

# Limit to specific wing
mempalace wake-up --wing myapp-project --aaak
```

### ChromaDB version conflicts

```bash
pip install "chromadb>=0.4.0" --upgrade
pip install mempalace --upgrade
```

---

## Environment Variables

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Required only for Haiku reranking (100% recall mode) |
| `MEMPALACE_PATH` | Override default palace path (`~/.mempalace/palace`) |
| `MEMPALACE_LOG_LEVEL` | Set to `DEBUG` for verbose output |

```bash
export MEMPALACE_PATH=/data/my-palace
export MEMPALACE_LOG_LEVEL=DEBUG
mempalace status
```

---

## Cost Reference

| Approach | Annual tokens | Annual cost |
|---|---|---|
| Paste everything | 19.5M (impossible) | — |
| LLM summaries | ~650K | ~$507/yr |
| MemPalace wake-up only | ~170 tokens | ~$0.70/yr |
| MemPalace + 5 searches/day | ~13,500 tokens | ~$10/yr |

MemPalace is entirely free. API costs above apply only if using Claude for optional reranking.
```
