```markdown
---
name: fireworks-tech-graph
description: Generate production-quality SVG+PNG technical diagrams from natural language using Claude Code. Supports 8 diagram types, 5 visual styles, and deep AI/Agent domain knowledge.
triggers:
  - generate a diagram
  - draw an architecture diagram
  - create a technical diagram
  - visualize my system
  - make a flowchart
  - draw a sequence diagram
  - generate SVG diagram
  - create architecture visualization
---

# fireworks-tech-graph

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Turn natural language descriptions into polished SVG + PNG technical diagrams. Supports 8 diagram types, 5 visual styles, and deep knowledge of AI/Agent system patterns (RAG, Mem0, Multi-Agent, Tool Call flows).

---

## Installation

```bash
# Via Claude Code skills
claude skills install fireworks-tech-graph

# Or clone directly
git clone https://github.com/yizhiyanhua-ai/fireworks-tech-graph.git ~/.claude/skills/fireworks-tech-graph
```

### System dependency (required for PNG export)

```bash
# macOS
brew install librsvg

# Ubuntu/Debian
sudo apt install librsvg2-bin

# Verify
rsvg-convert --version
```

---

## How It Works

```
User prompt → Skill classifies diagram type + style
           → Generates SVG with semantic shapes + arrows
           → Runs: rsvg-convert -w 1920 input.svg -o output.png
           → Reports file paths of .svg and .png
```

Output files are written to the current directory (or `--output` path).

---

## Basic Usage

### Trigger phrases (auto-detected)

```
generate diagram / draw diagram / create chart / visualize
architecture diagram / flowchart / sequence diagram / data flow
```

### Simple requests

```
Draw a RAG pipeline flowchart
Generate an Agentic Search architecture diagram
Create a tool call flow diagram
Visualize a microservices architecture
```

### Specify style

```
Draw a microservices architecture diagram, style 2 (dark terminal)
Draw a multi-agent collaboration diagram --style glassmorphism
Generate a Mem0 architecture diagram, blueprint style
```

### Specify output path

```
Generate a Mem0 architecture diagram, output to ~/Desktop/
Create a tool call flow diagram --output /tmp/diagrams/
```

---

## Visual Styles

| # | Name | Background | Font | Best For |
|---|------|------------|------|----------|
| 1 | **Flat Icon** *(default)* | `#ffffff` | Helvetica | Blogs, slides, docs |
| 2 | **Dark Terminal** | `#0f0f1a` | SF Mono / Fira Code | GitHub README, dev articles |
| 3 | **Blueprint** | `#0a1628` | Courier New | Architecture docs, engineering |
| 4 | **Notion Clean** | `#ffffff` | system-ui | Notion, Confluence, wikis |
| 5 | **Glassmorphism** | `#0d1117` gradient | Inter | Product sites, keynotes |

Reference files for each style live in `references/style-N-*.md` with exact color tokens and SVG patterns.

---

## Diagram Types

| Type | Description | Key Layout Rule |
|------|-------------|-----------------|
| **Architecture** | Services, components, cloud infra | Horizontal layers top→bottom |
| **Data Flow** | What data moves where | Label every arrow with data type |
| **Flowchart** | Decisions, process steps | Diamond = decision, top→bottom |
| **Agent Architecture** | LLM + tools + memory | 5-layer model: Input/Agent/Memory/Tool/Output |
| **Memory Architecture** | Mem0, MemGPT-style | Separate read/write paths, memory tiers |
| **Sequence** | API call chains, time-ordered | Vertical lifelines, horizontal messages |
| **Comparison** | Feature matrix, side-by-side | Column = system, row = attribute |
| **Mind Map** | Concept maps, radial | Central node, bezier branches |

---

## AI/Agent Domain Patterns (Built-in)

The skill has pre-loaded knowledge of these patterns — just name them:

```
RAG Pipeline         → Query → Embed → VectorSearch → Retrieve → LLM → Response
Agentic RAG          → RAG + Agent loop + Tool use
Agentic Search       → Query → Planner → [Search/Calc/Code] → Synthesizer
Mem0 Memory Layer    → Input → Memory Manager → [VectorDB + GraphDB] → Context
Agent Memory Types   → Sensory → Working → Episodic → Semantic → Procedural
Multi-Agent          → Orchestrator → [SubAgent×N] → Aggregator → Output
Tool Call Flow       → LLM → Tool Selector → Execution → Parser → LLM (loop)
```

Example prompts for each:

```
Generate a Mem0 memory architecture diagram with vector store, graph DB, KV store, and memory manager
Draw a Multi-Agent diagram: Orchestrator dispatches 3 SubAgents (search / compute / code), results aggregated
Visualize the Tool Call execution flow: LLM → Tool Selector → Execution → Parser → back to LLM
Compare Agentic RAG vs standard RAG in a feature matrix, Notion clean style
Draw the 5 agent memory types: Sensory, Working, Episodic, Semantic, Procedural
```

---

## Shape Vocabulary

Shapes carry semantic meaning consistently across all styles:

| Concept | Shape |
|---------|-------|
| User / Human | Circle + body |
| LLM / Model | Rounded rect, double border, ⚡ |
| Agent / Orchestrator | Hexagon |
| Memory (short-term) | Dashed-border rounded rect |
| Memory (long-term) | Solid cylinder |
| Vector Store | Cylinder with inner rings |
| Graph DB | 3-circle cluster |
| Tool / Function | Rect with ⚙ |
| API / Gateway | Hexagon (single border) |
| Queue / Stream | Horizontal pipe/tube |
| Document / File | Folded-corner rect |
| Browser / UI | Rect with 3-dot titlebar |
| Decision | Diamond |
| External Service | Dashed-border rect |

---

## Arrow Semantics

| Flow Type | Stroke | Dash | Meaning |
|-----------|--------|------|---------|
| Primary data flow | 2px solid | — | Main request/response |
| Control / trigger | 1.5px solid | — | System A triggers B |
| Memory read | 1.5px solid | — | Retrieve from store |
| Memory write | 1.5px | `5,3` | Write/store operation |
| Async / event | 1.5px | `4,2` | Non-blocking |
| Feedback / loop | 1.5px curved | — | Iterative reasoning |

---

## Example Prompts by Scenario

### AI/Agent Systems

```
Draw a microservices architecture: Client → API Gateway → [User Service / Order Service / Payment Service] → PostgreSQL + Redis
Generate a data pipeline: Kafka → Spark → S3 → Athena, blueprint style
Draw a Kubernetes deployment: Ingress → Service → [Pod × 3] → ConfigMap + PersistentVolume
```

### API & Sequence Flows

```
Draw an OAuth2 authorization code flow sequence diagram: User → Client → Auth Server → Resource Server
Draw the ChatGPT Plugin call sequence diagram
```

### Decision & Process Flows

```
Draw a pre-launch QA flowchart: Code Review → Security Scan → Performance Test → Manual Approval → Deploy
Generate a feature comparison matrix: RAG vs Fine-tuning vs Prompt Engineering
```

### Concept Maps

```
Visualize the LLM application tech stack: foundation model → SDK → app framework → deployment
Draw an AI Agent capability map: Perception / Memory / Reasoning / Action / Learning
```

---

## SVG Generation Guidelines

When generating SVG diagrams, follow these rules:

### Canvas & Layout

```
viewBox: "0 0 1600 900"   ← 16:9 default
preserveAspectRatio: "xMidYMid meet"
Padding: 60px all sides
Layer spacing: 140px vertical between swim lanes
Node spacing: 180px horizontal minimum
```

### Style 1 — Flat Icon (default) key tokens

```svg
<!-- Background -->
<rect width="1600" height="900" fill="#ffffff"/>

<!-- Node: LLM (double border) -->
<rect x="200" y="200" width="160" height="60" rx="8"
      fill="#EEF2FF" stroke="#6366F1" stroke-width="2"/>
<rect x="204" y="204" width="152" height="52" rx="6"
      fill="none" stroke="#6366F1" stroke-width="1" opacity="0.5"/>

<!-- Node: Agent (hexagon) -->
<polygon points="380,200 420,180 460,200 460,240 420,260 380,240"
         fill="#F0FDF4" stroke="#22C55E" stroke-width="2"/>

<!-- Primary arrow -->
<line x1="360" y1="230" x2="380" y2="230"
      stroke="#6366F1" stroke-width="2" marker-end="url(#arrowhead)"/>

<!-- Async arrow (dashed) -->
<line x1="360" y1="230" x2="380" y2="230"
      stroke="#94A3B8" stroke-width="1.5" stroke-dasharray="4,2"
      marker-end="url(#arrowhead-gray)"/>
```

### Style 2 — Dark Terminal key tokens

```svg
<rect width="1600" height="900" fill="#0f0f1a"/>
<!-- Nodes use: fill="#1a1a2e", stroke="#00ff88" or "#ff6b6b" or "#4ecdc4" -->
<!-- Text: fill="#e2e8f0", font-family="'SF Mono', 'Fira Code', monospace" -->
<!-- Primary arrows: stroke="#00ff88" -->
```

### Style 3 — Blueprint key tokens

```svg
<rect width="1600" height="900" fill="#0a1628"/>
<!-- Grid lines: stroke="#1e3a5f" stroke-width="1" opacity="0.4" -->
<!-- Nodes: fill="#0d2137", stroke="#00bcd4" stroke-width="1.5" -->
<!-- Text: fill="#b0c4de", font-family="'Courier New', monospace" -->
<!-- Arrows: stroke="#00bcd4" -->
```

### Style 5 — Glassmorphism key tokens

```svg
<defs>
  <linearGradient id="bg" x1="0%" y1="0%" x2="100%" y2="100%">
    <stop offset="0%" style="stop-color:#0d1117"/>
    <stop offset="100%" style="stop-color:#161b22"/>
  </linearGradient>
  <filter id="blur"><feGaussianBlur stdDeviation="4"/></filter>
</defs>
<rect width="1600" height="900" fill="url(#bg)"/>
<!-- Glass card: fill="rgba(255,255,255,0.05)", stroke="rgba(255,255,255,0.1)"
     backdrop blur via filter, border-radius 12px -->
```

### Arrowhead defs (include in every SVG)

```svg
<defs>
  <marker id="arrowhead" markerWidth="10" markerHeight="7"
          refX="9" refY="3.5" orient="auto">
    <polygon points="0 0, 10 3.5, 0 7" fill="#6366F1"/>
  </marker>
  <marker id="arrowhead-gray" markerWidth="10" markerHeight="7"
          refX="9" refY="3.5" orient="auto">
    <polygon points="0 0, 10 3.5, 0 7" fill="#94A3B8"/>
  </marker>
</defs>
```

---

## PNG Export Command

After generating the SVG file, always run this to produce the PNG:

```bash
rsvg-convert -w 1920 output.svg -o output.png
```

For custom dimensions:

```bash
# 2× retina width
rsvg-convert -w 3840 output.svg -o output@2x.png

# Fixed height
rsvg-convert -h 1080 output.svg -o output-1080.png

# Both dimensions (may distort — use only if aspect ratio matches)
rsvg-convert -w 1920 -h 1080 output.svg -o output.png
```

---

## File Naming Convention

```
{topic}-{type}-{style}.svg
{topic}-{type}-{style}.png

Examples:
  mem0-memory-architecture-dark.svg
  mem0-memory-architecture-dark.png
  rag-pipeline-flowchart-flat.svg
  multi-agent-architecture-glass.png
```

---

## Project File Structure

```
fireworks-tech-graph/
├── SKILL.md                      # Main skill definition
├── README.md                     # English docs
├── README.zh.md                  # Chinese docs
├── references/
│   ├── style-1-flat-icon.md      # Color tokens, SVG patterns
│   ├── style-2-dark-terminal.md
│   ├── style-3-blueprint.md
│   ├── style-4-notion-clean.md
│   ├── style-5-glassmorphism.md
│   └── icons.md                  # 40+ product icons + semantic shapes
└── assets/
    └── samples/                  # Sample PNG outputs per style
```

---

## Product Icon Coverage

When a node represents a known product, use its brand color as the node accent:

| Product | Brand Color |
|---------|-------------|
| OpenAI | `#00A67E` |
| Anthropic/Claude | `#D97757` |
| Google Gemini | `#4285F4` |
| Meta LLaMA | `#0668E1` |
| Pinecone | `#1A1A2E` + `#00FF88` |
| Weaviate | `#FA0050` |
| LangChain | `#1C3C3C` |
| Kafka | `#231F20` + `#FF6B35` |
| PostgreSQL | `#336791` |
| Redis | `#DC382D` |
| Kubernetes | `#326CE5` |
| AWS | `#FF9900` |
| GCP | `#4285F4` |
| Azure | `#0078D4` |

Full icon reference: `references/icons.md`

---

## Troubleshooting

### `rsvg-convert: command not found`

```bash
# macOS — install librsvg
brew install librsvg

# Ubuntu
sudo apt-get update && sudo apt-get install -y librsvg2-bin

# Check PATH
which rsvg-convert
```

### PNG is blank or all black

- Check SVG has explicit `width`/`height` or a valid `viewBox`
- Ensure `fill` colors are not `transparent` on the root `<rect>`
- Dark style backgrounds need explicit background rect before other elements

### Text appears as boxes / missing glyphs

- The skill uses only system-safe fonts: `Helvetica`, `Arial`, `Courier New`, `system-ui`, `monospace`
- Avoid Google Fonts or web fonts — `rsvg-convert` does not fetch external resources
- For emoji icons (⚡ ⚙), test with `rsvg-convert` — fall back to geometric SVG shapes if unsupported

### SVG renders fine but PNG is low resolution

```bash
# Increase width — 1920 is minimum, 3840 for retina
rsvg-convert -w 3840 input.svg -o output@2x.png
```

### Diagram is too cramped

- Increase canvas: change `viewBox="0 0 1600 900"` to `"0 0 2400 1200"`
- Increase node spacing: use 220px horizontal, 160px vertical between layers
- Add swim lane padding: 40px inside each lane boundary

### Style not applying

- Reference the style by number (1–5) or name: `flat`, `dark`, `blueprint`, `notion`, `glass`
- If ambiguous, default is Style 1 (Flat Icon)

---

## Quick Reference Card

```
STYLES:    1=flat  2=dark  3=blueprint  4=notion  5=glass
TYPES:     architecture | dataflow | flowchart | agent
           memory | sequence | comparison | mindmap
SHAPES:    circle=user  hexagon=agent  double-rect=LLM
           cylinder=storage  diamond=decision  pipe=queue
ARROWS:    solid=primary  dashed(5,3)=write  dashed(4,2)=async
EXPORT:    rsvg-convert -w 1920 input.svg -o output.png
CANVAS:    viewBox="0 0 1600 900"  padding=60px
```
```
