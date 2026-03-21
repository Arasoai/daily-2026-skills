```markdown
---
name: dontbesilent-business-diagnostics
description: Business diagnosis toolkit distilled from 12,307 tweets into Claude Code skills — commercial model diagnosis, benchmarking, content strategy, execution unblocking, and concept deconstruction.
triggers:
  - diagnose my business model
  - run business diagnosis
  - benchmark against competitors
  - I can't execute on my plans
  - deconstruct this business concept
  - analyze my content strategy
  - use dbs skill
  - commercial diagnosis
---

# dontbesilent Business Diagnostics (dbskill)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A business diagnosis toolkit distilled from 12,307 tweets into structured Claude Code skills. Provides six diagnostic tools covering commercial model validation, competitor benchmarking, content strategy, execution unblocking, and Wittgenstein-style concept deconstruction. Knowledge base includes 4,176 structured knowledge atoms and 10 methodology documents.

---

## Installation

```bash
# Via npx (recommended)
npx skills add dontbesilent2025/dbskill

# Manual install
git clone https://github.com/dontbesilent2025/dbskill.git /tmp/dbskill \
  && cp -r /tmp/dbskill/skills/dbs* ~/.claude/skills/ \
  && rm -rf /tmp/dbskill
```

After installation, skills are available in Claude Code via `/dbs` commands.

---

## Available Skills

| Command | Purpose |
|---|---|
| `/dbs` | Main entry — auto-routes to the right tool |
| `/dbs-diagnosis` | Business model diagnosis. Dissolves problems, doesn't just answer them |
| `/dbs-benchmark` | Competitor benchmarking with five-layer noise filtering |
| `/dbs-content` | Content creation diagnosis via five-dimension detection |
| `/dbs-unblock` | Execution diagnosis using Adler framework |
| `/dbs-deconstruct` | Concept deconstruction via Wittgenstein-style audit |

### Recommended Workflow

```
/dbs-diagnosis   → Is the business model right?
      ↓
/dbs-benchmark   → Who to model after?
      ↓
/dbs-content     → How to produce content?
      ↓
/dbs-unblock     → Stuck? Can't execute?

/dbs-deconstruct → Deconstruct any concept at any time
```

Skills cross-recommend: if `/dbs-diagnosis` detects psychological blockers, it will suggest `/dbs-unblock`.

---

## Knowledge Base Structure

```
知识库/
├── 原子库/                          # Structured knowledge database
│   ├── atoms.jsonl                  # 4,176 knowledge atoms (full)
│   ├── atoms_2024Q4.jsonl           # Quarterly splits
│   ├── atoms_2025Q1.jsonl
│   └── README.md
│
├── Skill知识包/                      # Distilled methodology docs
│   ├── diagnosis_公理与诊断框架.md   # Diagnosis axioms & framework
│   ├── diagnosis_问题消解案例库.md   # Problem dissolution case library
│   ├── benchmark_对标方法论.md       # Benchmarking methodology
│   ├── benchmark_平台运营知识.md     # Platform operations knowledge
│   ├── content_内容创作方法论.md     # Content creation methodology
│   ├── content_平台特性与案例.md     # Platform characteristics & cases
│   ├── unblock_心理诊断框架.md       # Psychological diagnosis framework
│   ├── unblock_信号案例库.md         # Signal case library
│   ├── deconstruct_语言与概念框架.md # Language & concept framework
│   └── deconstruct_解构案例库.md     # Deconstruction case library
│
└── 高频概念词典.md                   # High-frequency concept dictionary
```

---

## Knowledge Atom Schema

Each atom extracted from tweets is structured as:

```json
{
  "id": "2024Q4_042",
  "knowledge": "判断一个生意能不能做，必要条件之一是你能不能说出这个产品的颜色",
  "original": "原始推文内容（≤200字）",
  "url": "https://x.com/dontbesilent/status/...",
  "date": "2024-10-01",
  "topics": ["商业模式与定价", "语言与思维"],
  "skills": ["dbs-diagnosis", "dbs-deconstruct"],
  "type": "anti-pattern",
  "confidence": "high"
}
```

### Field Reference

| Field | Values | Description |
|-------|--------|-------------|
| `knowledge` | string | Distilled knowledge point |
| `original` | string | Original tweet text (≤200 chars) |
| `topics` | array | 10 topic categories (multi-select) |
| `skills` | array | Associated skills |
| `type` | `principle` \| `method` \| `case` \| `anti-pattern` \| `insight` \| `tool` | Atom type |
| `confidence` | `high` \| `medium` \| `low` | Reliability score |

### Topic Categories

- 商业模式与定价 (Business model & pricing)
- 语言与思维 (Language & thinking)
- 心理与执行力 (Psychology & execution)
- 内容创作 (Content creation)
- 平台运营 (Platform operations)
- 对标与竞争 (Benchmarking & competition)
- 产品设计 (Product design)
- 用户行为 (User behavior)
- 创业与增长 (Startup & growth)
- 概念解构 (Concept deconstruction)

---

## Using the Knowledge Base Without Installing Skills

### Scenario 1: Add Business Diagnosis to Your AI's System Prompt

```python
# Read the diagnosis axioms doc and inject into system prompt
with open("知识库/Skill知识包/diagnosis_公理与诊断框架.md", "r") as f:
    diagnosis_framework = f.read()

system_prompt = f"""
You are a business diagnostics assistant.
Use the following framework for all diagnoses:

{diagnosis_framework}
"""
```

### Scenario 2: RAG Knowledge Base from atoms.jsonl

```python
import json

def load_atoms(path="知识库/原子库/atoms.jsonl", filters=None):
    atoms = []
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            atom = json.loads(line.strip())
            if filters:
                # Filter by topic
                if "topic" in filters:
                    if not any(t in atom["topics"] for t in filters["topic"]):
                        continue
                # Filter by type
                if "type" in filters and atom["type"] != filters["type"]:
                    continue
                # Filter by skill
                if "skill" in filters:
                    if filters["skill"] not in atom.get("skills", []):
                        continue
                # Filter by confidence
                if "confidence" in filters and atom["confidence"] != filters["confidence"]:
                    continue
            atoms.append(atom)
    return atoms

# Load only high-confidence anti-patterns for diagnosis
diagnosis_antipatterns = load_atoms(filters={
    "skill": "dbs-diagnosis",
    "type": "anti-pattern",
    "confidence": "high"
})

# Load psychology & execution atoms for unblock skill
unblock_atoms = load_atoms(filters={
    "topic": ["心理与执行力"]
})
print(f"Found {len(unblock_atoms)} execution-related atoms")
# → Found 296 execution-related atoms
```

### Scenario 3: Vector DB Ingestion for Semantic Search

```python
import json
from typing import Generator

def atom_to_document(atom: dict) -> dict:
    """Convert knowledge atom to vector DB document."""
    return {
        "id": atom["id"],
        "text": atom["knowledge"],  # Embed the distilled knowledge
        "metadata": {
            "original": atom.get("original", ""),
            "url": atom.get("url", ""),
            "date": atom.get("date", ""),
            "topics": atom["topics"],
            "skills": atom["skills"],
            "type": atom["type"],
            "confidence": atom["confidence"],
        }
    }

def stream_atoms(path="知识库/原子库/atoms.jsonl") -> Generator:
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            if line.strip():
                yield atom_to_document(json.loads(line))

# Example: ingest into any vector store
documents = list(stream_atoms())
print(f"Total atoms for ingestion: {len(documents)}")
# → Total atoms for ingestion: 4176

# Filter to cases only (~700+ real business cases)
cases = [d for d in documents if d["metadata"]["type"] in ("case", "anti-pattern")]
print(f"Business cases available: {len(cases)}")
```

### Scenario 4: Build a Chatbot with Methodology + RAG

```python
import json
import os

def build_chatbot_context(skill: str) -> str:
    """
    Build a system prompt for a specific skill's chatbot.
    Combines methodology doc + relevant high-confidence atoms.
    """
    base_dir = "知识库"
    
    # Map skill to knowledge pack files
    skill_docs = {
        "dbs-diagnosis": [
            f"{base_dir}/Skill知识包/diagnosis_公理与诊断框架.md",
            f"{base_dir}/Skill知识包/diagnosis_问题消解案例库.md",
        ],
        "dbs-benchmark": [
            f"{base_dir}/Skill知识包/benchmark_对标方法论.md",
            f"{base_dir}/Skill知识包/benchmark_平台运营知识.md",
        ],
        "dbs-unblock": [
            f"{base_dir}/Skill知识包/unblock_心理诊断框架.md",
            f"{base_dir}/Skill知识包/unblock_信号案例库.md",
        ],
        "dbs-content": [
            f"{base_dir}/Skill知识包/content_内容创作方法论.md",
            f"{base_dir}/Skill知识包/content_平台特性与案例.md",
        ],
        "dbs-deconstruct": [
            f"{base_dir}/Skill知识包/deconstruct_语言与概念框架.md",
            f"{base_dir}/Skill知识包/deconstruct_解构案例库.md",
        ],
    }
    
    docs_content = []
    for doc_path in skill_docs.get(skill, []):
        if os.path.exists(doc_path):
            with open(doc_path, "r", encoding="utf-8") as f:
                docs_content.append(f.read())
    
    return "\n\n---\n\n".join(docs_content)


# Usage with any LLM
system_prompt = build_chatbot_context("dbs-diagnosis")
# Pass system_prompt to your LLM API call
```

### Scenario 5: Filter Atoms by Research Topic

```python
import json
from collections import Counter

def analyze_topic_distribution(path="知识库/原子库/atoms.jsonl"):
    topic_counter = Counter()
    type_counter = Counter()
    
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            atom = json.loads(line.strip())
            for topic in atom["topics"]:
                topic_counter[topic] += 1
            type_counter[atom["type"]] += 1
    
    print("=== Topic Distribution ===")
    for topic, count in topic_counter.most_common():
        print(f"  {topic}: {count}")
    
    print("\n=== Type Distribution ===")
    for atype, count in type_counter.most_common():
        print(f"  {atype}: {count}")

analyze_topic_distribution()

# Get all atoms for a specific research area
def get_topic_atoms(topic: str, min_confidence: str = "medium"):
    confidence_rank = {"low": 0, "medium": 1, "high": 2}
    min_rank = confidence_rank[min_confidence]
    
    results = []
    with open("知识库/原子库/atoms.jsonl", "r", encoding="utf-8") as f:
        for line in f:
            atom = json.loads(line.strip())
            if topic in atom["topics"]:
                if confidence_rank.get(atom["confidence"], 0) >= min_rank:
                    results.append(atom)
    return results

pricing_atoms = get_topic_atoms("商业模式与定价", min_confidence="high")
print(f"High-confidence pricing atoms: {len(pricing_atoms)}")
```

---

## Diagnosis Framework Quick Reference

The `/dbs-diagnosis` skill operates on **6 core axioms** with a **dissolution funnel** — it aims to dissolve problems at the root rather than patch symptoms.

Key diagnostic questions it applies:
1. Can you describe the product's color? (Specificity test)
2. Is this a real problem or a definition problem?
3. Is the blocker external or psychological?
4. Who is already solving this successfully? (Benchmark signal)
5. What would have to be true for this to work?
6. Are you answering the wrong question?

## Benchmarking Five-Layer Noise Filter

The `/dbs-benchmark` skill applies five filters to eliminate false benchmarks:

1. **Scale filter** — Remove companies at different scale stages
2. **Context filter** — Remove different market/cultural contexts  
3. **Timing filter** — Remove companies in different growth phases
4. **Survivorship filter** — Include failed attempts, not just successes
5. **Causation filter** — Separate what caused success from what accompanied it

## Execution Unblocking (Adler Framework)

The `/dbs-unblock` skill uses Alfred Adler's psychology framework:
- All problems are ultimately interpersonal relationship problems
- "Can't do" usually means "won't do" — find the hidden secondary gain
- Separate the task from the person's identity
- Identify the specific moment execution breaks down

---

## Troubleshooting

**Skills not appearing after install**
```bash
# Verify installation location
ls ~/.claude/skills/ | grep dbs

# Re-run manual install
cp -r /tmp/dbskill/skills/dbs* ~/.claude/skills/
```

**Knowledge base files not found**
```bash
# Check you cloned the full repo including 知识库/ directory
ls 知识库/原子库/atoms.jsonl
ls "知识库/Skill知识包/"
```

**atoms.jsonl parse errors**
```python
# Handle encoding and empty lines defensively
import json

def safe_load_atoms(path):
    atoms = []
    with open(path, "r", encoding="utf-8") as f:
        for i, line in enumerate(f):
            line = line.strip()
            if not line:
                continue
            try:
                atoms.append(json.loads(line))
            except json.JSONDecodeError as e:
                print(f"Parse error on line {i+1}: {e}")
    return atoms
```

**Skill routes to wrong tool**
Use specific commands instead of `/dbs` router:
```
/dbs-diagnosis  # for business model issues
/dbs-unblock    # for execution/psychological blockers
/dbs-benchmark  # for competitive analysis
```

---

## License

[CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)

- Personal use, learning, research, non-commercial projects: no attribution required
- Public derivative works (articles, tools, courses): please credit the source
- Commercial use: requires separate authorization — contact the author

Author: [dontbesilent](https://x.com/dontbesilent)
```
