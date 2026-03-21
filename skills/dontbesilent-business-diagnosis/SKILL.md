```markdown
---
name: dontbesilent-business-diagnosis
description: Business diagnosis toolkit for Claude Code — routes to diagnosis, benchmark, content, unblock, and deconstruct skills extracted from 12,307 tweets into 4,176 structured knowledge atoms.
triggers:
  - diagnose my business model
  - benchmark against competitors
  - help me create content strategy
  - I'm stuck and can't execute
  - deconstruct this business concept
  - run dbs diagnosis
  - business model analysis
  - why can't I get unblocked
---

# dontbesilent Business Diagnosis Toolkit

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A Claude Code skill suite distilled from 12,307 tweets into 4,176 structured knowledge atoms. Provides business diagnosis, competitor benchmarking, content strategy, execution unblocking, and concept deconstruction — all as slash commands.

---

## Installation

```bash
npx skills add dontbesilent2025/dbskill
```

Or manually:

```bash
git clone https://github.com/dontbesilent2025/dbskill.git /tmp/dbskill \
  && cp -r /tmp/dbskill/skills/dbs* ~/.claude/skills/ \
  && rm -rf /tmp/dbskill
```

After installation, open Claude Code and type `/dbs` to start.

---

## Skills Overview

| Command | Purpose |
|---|---|
| `/dbs` | Main router — auto-routes to the right tool |
| `/dbs-diagnosis` | Business model diagnosis — dissolves problems instead of answering them |
| `/dbs-benchmark` | Competitor analysis — 5-layer filter to eliminate noise |
| `/dbs-content` | Content creation diagnosis — 5-dimension detection |
| `/dbs-unblock` | Execution diagnosis — Adler framework |
| `/dbs-deconstruct` | Concept breakdown — Wittgenstein-style audit |

### Workflow

```
/dbs-diagnosis    → Is the business model correct?
       ↓
/dbs-benchmark    → Who should I model after?
       ↓
/dbs-content      → How do I do the content?
       ↓
/dbs-unblock      → Why can't I get moving?

/dbs-deconstruct  → Use anytime to audit concepts
```

Skills auto-recommend next steps. For example, if `/dbs-diagnosis` detects psychological blockers, it will suggest `/dbs-unblock`.

---

## Usage Examples

### Route Automatically

```
/dbs I have a SaaS product but sales are flat after 6 months
```

The router reads the context and forwards to `/dbs-diagnosis`.

### Business Model Diagnosis

```
/dbs-diagnosis

My product: online course platform for indie developers
Revenue: $3k MRR, flat for 4 months
Traffic: growing
Churn: ~30% monthly
```

The skill applies the 6-axiom dissolution funnel — it does **not** give generic advice. It identifies which axiom is violated and dissolves the problem at its root.

**Core axioms used internally:**

1. Can you describe the product's color? (Specificity test)
2. Who loses money if your product disappears? (Value anchoring)
3. Is the problem a business problem or a psychology problem?
4. Are you selling a vitamin or a painkiller?
5. Can the customer explain what they bought to a friend?
6. Does the price feel like a bargain or a compromise?

### Benchmark Analysis

```
/dbs-benchmark

I run a newsletter for B2B SaaS founders, 2,400 subscribers, 42% open rate.
Who should I be benchmarking against?
```

Applies 5-layer filtering:
1. Same audience specificity
2. Same monetization model
3. Same growth stage
4. Same content format
5. Same distribution channel

Returns only high-signal benchmarks, filtered for noise.

### Content Diagnosis

```
/dbs-content

Here's my last 5 posts: [paste posts]
They get impressions but zero engagement.
```

5-dimension detection:
- Hook quality
- Specificity of claim
- Reader self-recognition
- Call to action clarity
- Platform fit

### Execution Unblocking

```
/dbs-unblock

I know exactly what I need to do but I haven't done it in 3 weeks.
```

Uses Adler framework — distinguishes between "can't do" and "won't do." Identifies the real blocker (courage deficit, goal misalignment, or environmental friction) and gives a single actionable next step.

### Concept Deconstruction

```
/dbs-deconstruct

Everyone keeps telling me I need "product-market fit."
What does that actually mean?
```

Wittgenstein-style audit: strips the concept to its observable behaviors, removes jargon, returns a falsifiable definition you can act on.

---

## Knowledge Base

The knowledge base is fully open and modular. You can use any part without installing the full skill suite.

### Directory Structure

```
知识库/
├── 原子库/                      # Structured knowledge database
│   ├── atoms.jsonl              # 4,176 knowledge atoms (full)
│   ├── atoms_2024Q4.jsonl       # Quarterly splits
│   ├── atoms_2025Q1.jsonl
│   └── README.md
│
├── Skill知识包/                  # Distilled methodology docs
│   ├── diagnosis_公理与诊断框架.md
│   ├── diagnosis_问题消解案例库.md
│   ├── benchmark_对标方法论.md
│   ├── benchmark_平台运营知识.md
│   ├── content_内容创作方法论.md
│   ├── content_平台特性与案例.md
│   ├── unblock_心理诊断框架.md
│   ├── unblock_信号案例库.md
│   ├── deconstruct_语言与概念框架.md
│   └── deconstruct_解构案例库.md
│
└── 高频概念词典.md
```

### Knowledge Atom Schema

Each atom is a JSON line:

```json
{
  "id": "2024Q4_042",
  "knowledge": "判断一个生意能不能做，必要条件之一是你能不能说出这个产品的颜色",
  "original": "判断一个生意能不能做，必要条件之一是你能不能说出这个产品的颜色...",
  "url": "https://x.com/dontbesilent/status/...",
  "date": "2024-10-01",
  "topics": ["商业模式与定价", "语言与思维"],
  "skills": ["dbs-diagnosis", "dbs-deconstruct"],
  "type": "anti-pattern",
  "confidence": "high"
}
```

| Field | Values |
|---|---|
| `type` | `principle` / `method` / `case` / `anti-pattern` / `insight` / `tool` |
| `confidence` | `high` / `medium` / `low` |
| `topics` | 10 topic categories, multi-select |
| `skills` | Which skills reference this atom |

### Using the Knowledge Base Directly

**Add business diagnosis to any AI system prompt:**

```python
# Read the axioms framework and inject into system prompt
with open("知识库/Skill知识包/diagnosis_公理与诊断框架.md") as f:
    axioms = f.read()

system_prompt = f"""
You are a business diagnosis assistant.
Use the following framework:

{axioms}
"""
```

**Build a RAG knowledge base:**

```python
import json

atoms = []
with open("知识库/原子库/atoms.jsonl") as f:
    for line in f:
        atoms.append(json.loads(line))

# Filter by skill
diagnosis_atoms = [
    a for a in atoms
    if "dbs-diagnosis" in a["skills"]
    and a["confidence"] == "high"
]

# Filter for cases and anti-patterns only
cases = [
    a for a in atoms
    if a["type"] in ("case", "anti-pattern")
]
# ~700+ real business examples

# Filter by topic
execution_atoms = [
    a for a in atoms
    if "心理与执行力" in a["topics"]
]
# Returns ~296 atoms
```

**Query atoms for a chatbot RAG pipeline:**

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2")

# Build embeddings
texts = [a["knowledge"] for a in atoms]
embeddings = model.encode(texts)

def retrieve(query: str, top_k: int = 5):
    q_emb = model.encode([query])
    scores = np.dot(embeddings, q_emb.T).squeeze()
    top_idx = scores.argsort()[::-1][:top_k]
    return [atoms[i] for i in top_idx]

results = retrieve("我的产品有流量但没转化")
for r in results:
    print(r["knowledge"])
    print(r["url"])
    print()
```

---

## Common Patterns

### Pattern: Problem → Dissolution (not solution)

The core philosophy: `/dbs-diagnosis` does **not** answer "how do I fix this?" It first checks whether the problem is real, correctly framed, and at the right level of abstraction. Most business problems dissolve when properly described.

```
User: "My conversion rate is 2%, how do I improve it?"

/dbs-diagnosis response pattern:
1. Is 2% low for your specific funnel stage and traffic source? (Reality check)
2. What is the unit economics at 2%? Is it profitable? (Reframing)
3. Is the problem conversion, or is it traffic quality? (Level shift)
→ Often the problem dissolves: 2% is fine, the real issue is traffic cost.
```

### Pattern: Benchmark Noise Filtering

```
Wrong benchmark: "I want to grow like Morning Brew"
→ Different audience specificity, different monetization, different stage

Right benchmark (after 5-layer filter):
→ Newsletter at same subscriber count, same niche, same revenue model
→ 3 real examples from atoms with type: "case"
```

### Pattern: Unblock Signal Detection

```
Signal: "I know what to do but keep postponing"
→ NOT a time management problem
→ Adler diagnosis: Which task specifically? What happens if you do it?
→ Usually reveals: fear of the result, not the task itself
```

---

## Troubleshooting

**`/dbs` command not found after installation**

```bash
# Verify skills were copied correctly
ls ~/.claude/skills/ | grep dbs

# If missing, reinstall
git clone https://github.com/dontbesilent2025/dbskill.git /tmp/dbskill \
  && cp -r /tmp/dbskill/skills/dbs* ~/.claude/skills/ \
  && rm -rf /tmp/dbskill
```

**Skill runs but knowledge packages aren't loading**

The skills read knowledge packages from relative paths. Ensure the full repo structure is preserved:

```bash
# Check that knowledge packages exist alongside skills
ls ~/.claude/skills/dbs-diagnosis/
# Should include: SKILL.md + references to 知识库/ content
```

**Using knowledge base outside Claude Code**

The `.jsonl` atoms and `.md` knowledge packages are standalone — no Claude Code dependency. Load them directly into any LLM pipeline using the Python examples above.

**Atoms have Chinese text — will they work in English contexts?**

Yes. The `knowledge` field contains the distilled insight. For English RAG, use a multilingual embedding model (e.g., `paraphrase-multilingual-MiniLM-L12-v2`) or translate the `knowledge` field at index time.

---

## License

[CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)

- Personal use, learning, research, non-commercial projects: no attribution required
- Public derivative works (articles, tools, courses): credit the source
- Commercial use: requires separate authorization — contact the author

Author: [dontbesilent](https://x.com/dontbesilent)
```
