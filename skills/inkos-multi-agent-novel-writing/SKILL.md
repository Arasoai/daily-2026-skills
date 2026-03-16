---
name: inkos-multi-agent-novel-writing
description: Multi-agent CLI system for autonomous novel writing, auditing, and revision with human review gates
triggers:
  - write novel with AI agents
  - automated fiction writing pipeline
  - multi-agent novel production
  - inkos novel writing
  - AI novel auditing and revision
  - autonomous chapter generation
  - novel writing CLI agent
  - inkos book create write audit
---

# InkOS — Multi-Agent Novel Writing System

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection

InkOS is a TypeScript CLI system that orchestrates multiple AI agents to autonomously write, audit, and revise novels. Agents handle chapter writing, quality auditing (33 dimensions), revision, AIGC detection, and style mimicry — with human review gates between pipeline stages.

## Installation

```bash
# Global install
npm install -g @actalk/inkos

# Or run via npx
npx @actalk/inkos --help

# Verify
inkos --version
```

## Core Concepts

| Agent | Role |
|-------|------|
| **Writer** | Generates chapter content using genre rules, style guides, and cross-chapter memory |
| **Auditor** | Evaluates chapters across 33 dimensions including AIGC detection, plot consistency, pacing |
| **Reviser** | Applies `spot-fix` or `rewrite` patches based on audit findings |
| **Architect** | Creates outlines, book rules, character sheets |
| **Scheduler** | Orchestrates the pipeline with retry logic, daily limits, parallel book processing |

## Quick Start

```bash
# 1. Configure LLM provider
inkos config set llm.provider openai
inkos config set llm.model gpt-4o

# 2. Create a new book
inkos book create --title "吞天魔帝" --genre xuanhuan

# 3. Generate outline and chapter plan
inkos outline 吞天魔帝

# 4. Write next chapter
inkos write next 吞天魔帝

# 5. Audit the latest chapter
inkos audit 吞天魔帝 --chapter latest

# 6. Revise based on audit
inkos revise 吞天魔帝 --chapter latest --mode spot-fix

# 7. Run full pipeline (write → audit → revise loop)
inkos run 吞天魔帝
```

## Project Structure

After `inkos book create`, the project directory looks like:

```
my-novel/
├── story/
│   ├── outline.md              # Story arc and chapter plan
│   ├── characters.md           # Character sheets
│   ├── world.md                # World-building rules
│   ├── chapter_summaries.md    # Per-chapter summaries (cross-chapter memory)
│   ├── subplot_board.md        # A/B/C subplot tracking
│   ├── emotional_arcs.md       # Per-character emotional arc tracking
│   ├── character_matrix.md     # Character interaction + info boundary matrix
│   └── parent_canon.md         # Spinoff only: imported canon constraints
├── chapters/
│   ├── ch001.md
│   └── ch002.md
├── audit/
│   └── ch001_audit.json        # Structured audit results
├── style_profile.json          # Statistical style fingerprint
├── style_guide.md              # LLM-generated qualitative style guide
├── book_rules.md               # Per-book custom rules and prohibitions
├── detection_history.json      # AIGC detection history
└── inkos.config.json           # Book-level config
```

## Configuration

### Global config (`~/.inkos/config.json`)

```json
{
  "llm": {
    "provider": "openai",
    "model": "gpt-4o",
    "temperature": 0.7,
    "maxTokens": 4096
  },
  "scheduler": {
    "intervalMinutes": 15,
    "dailyChapterLimit": 10,
    "parallelBooks": 2,
    "retryOnAuditFail": true
  },
  "webhook": {
    "url": "https://your-endpoint.com/hooks/inkos",
    "secret": "your-hmac-secret",
    "events": ["chapter-complete", "audit-failed", "pipeline-error"]
  }
}
```

### Book rules (`book_rules.md`)

```markdown
# Book Rules: 吞天魔帝

## Prohibitions
- Do not reference the "Nine Heaven Realm" until chapter 50
- The protagonist must not kill humans before chapter 20

## additionalAuditDimensions
- 主角性格一致性: 检查主角行为是否符合其既定性格设定
- 战力体系自洽: 新出现的力量不得违反已建立的数值体系

## fatiguedWords
- 震惊
- 不可思议
- 前所未有
```

## Key Commands

### Book Management

```bash
inkos book create --title "书名" --genre xuanhuan   # xuanhuan|xianxia|urban|horror|general
inkos book list
inkos book status 吞天魔帝
inkos book delete 吞天魔帝
```

### Writing Pipeline

```bash
inkos write next 吞天魔帝                    # Write next chapter
inkos write chapter 吞天魔帝 --num 5         # Write specific chapter
inkos audit 吞天魔帝 --chapter latest        # Audit latest chapter
inkos audit 吞天魔帝 --chapter 3 --temp 0    # Audit ch3, temperature locked to 0
inkos revise 吞天魔帝 --chapter latest --mode spot-fix   # Targeted fix
inkos revise 吞天魔帝 --chapter latest --mode rewrite    # Full rewrite
inkos revise 吞天魔帝 --chapter latest --mode anti-detect # AIGC removal pass
inkos run 吞天魔帝                           # Full autonomous pipeline
inkos daemon start                           # Background scheduler
inkos daemon stop
```

### Genre Management

```bash
inkos genre list
inkos genre show xuanhuan
inkos genre copy xuanhuan          # Copy to project for customization
inkos genre create wuxia --name 武侠
```

### Style Mimicry

```bash
inkos style analyze reference.txt               # Extract style fingerprint
inkos style import reference.txt 吞天魔帝 --name 某作者  # Import style to book
```

Produces `style_profile.json` (statistical) and `style_guide.md` (qualitative). Both are automatically injected into Writer and Auditor prompts.

### Spinoff / Canon Import

```bash
inkos book create --title "烈焰前传" --genre xuanhuan
inkos import canon 烈焰前传 --from 吞天魔帝   # Import parent canon constraints
inkos write next 烈焰前传                     # Writer auto-loads parent_canon.md
```

Spinoff auditor activates 4 additional dimensions automatically when `parent_canon.md` exists:
- Parent canon event conflicts
- Future information leakage
- Cross-book world rule consistency  
- Spinoff foreshadowing isolation

### AIGC Detection

```bash
inkos detect 吞天魔帝 --chapter latest       # Detect single chapter
inkos detect 吞天魔帝 --all                  # Full book scan
inkos detect --stats                         # View detection history stats
```

Configure external detection API in `inkos.config.json`:

```json
{
  "detection": {
    "provider": "gptzero",
    "endpoint": "https://api.gptzero.me/v2/predict/text",
    "threshold": 0.7
  }
}
```

## Post-Write Validator (Zero LLM Cost)

11 deterministic rules run after every chapter write, before LLM audit:

```typescript
// These fire automatically — no configuration needed
// When error-level violations found, spot-fix triggers immediately

const VALIDATOR_RULES = {
  bannedPatterns: ['不是……而是……'],
  bannedPunctuation: ['——'],
  transitionWordDensity: { words: ['仿佛','忽然','竟然'], maxPer3000: 1 },
  fatiguedWords: { maxPerWord: 1 },
  metanarrative: true,           // Blocks "作者说教" style phrases
  reportTerminology: true,       // Blocks analytical framework terms
  collectiveReaction: true,      // Blocks "全场震惊" clichés
  consecutiveLE: { maxSentences: 3 },  // Consecutive sentences with 了
  longParagraph: { maxChars: 300, maxCount: 1 },
  bookTaboos: true               // Reads from book_rules.md
};
```

## Audit Dimensions (33 Total)

The auditor scores each chapter across these categories (scores trigger auto-revise when critical):

| Range | Category |
|-------|----------|
| 1–10 | Core quality: consistency, pacing, character, world-building |
| 11–20 | Genre-specific: power systems, plot mechanics, tension |
| 21–26 | Anti-AI: AIGC markers, formulaic structures, style fingerprint |
| 27 | Sensitive content |
| 28–31 | Subplot, arc, rhythm dimensions (all 5 genres) |
| 32 | Reader expectation management |
| 33 | Outline deviation detection |

## TypeScript Integration

InkOS exposes a programmatic API for embedding in your own TypeScript projects:

```typescript
import { InkOS, BookConfig, PipelineOptions } from '@actalk/inkos';

const inkos = new InkOS({
  llm: { provider: 'openai', model: 'gpt-4o' },
  scheduler: { intervalMinutes: 15, dailyChapterLimit: 10 }
});

// Create and configure a book
const book = await inkos.createBook({
  title: '吞天魔帝',
  genre: 'xuanhuan',
  targetChapters: 100,
  wordsPerChapter: 3000
} satisfies BookConfig);

// Run one pipeline cycle
const result = await inkos.runCycle(book.id, {
  autoRevise: true,
  reviseMode: 'spot-fix',
  auditTemperature: 0
} satisfies PipelineOptions);

console.log(result.chapter.title);     // Chapter title
console.log(result.audit.score);       // Overall audit score
console.log(result.audit.criticals);   // Critical issues found
console.log(result.revised);           // Whether revision was applied
```

### Writing a Single Chapter Programmatically

```typescript
import { WriterAgent, AuditorAgent, ReviserAgent } from '@actalk/inkos/agents';
import { loadBook, loadMemoryContext } from '@actalk/inkos/context';

const book = await loadBook('./my-novel');
const memory = await loadMemoryContext(book);  // Loads all truth files

// Write
const writer = new WriterAgent({ temperature: 0.8 });
const chapter = await writer.write({
  book,
  memory,
  chapterNum: 42,
  styleGuide: book.styleGuide   // Injected automatically if present
});

// Audit (temperature: 0 for determinism)
const auditor = new AuditorAgent({ temperature: 0 });
const audit = await auditor.audit({ book, chapter, memory });

// Conditionally revise
if (audit.criticals.length > 0) {
  const reviser = new ReviserAgent();
  const revised = await reviser.revise({
    chapter,
    audit,
    mode: 'spot-fix',  // Only touch flagged sentences
    validateAIMarkers: true  // Discard revision if AI markers increase
  });
  await revised.save();
}
```

### Custom Webhook Handler

```typescript
import express from 'express';
import { verifyWebhookSignature } from '@actalk/inkos/webhook';

const app = express();
app.use(express.json());

app.post('/hooks/inkos', (req, res) => {
  const valid = verifyWebhookSignature(
    req.body,
    req.headers['x-inkos-signature'] as string,
    process.env.INKOS_WEBHOOK_SECRET!
  );
  if (!valid) return res.sendStatus(401);

  const { event, book, chapter, audit } = req.body;

  switch (event) {
    case 'chapter-complete':
      console.log(`✅ ${book.title} ch${chapter.num} written`);
      break;
    case 'audit-failed':
      console.log(`❌ Audit failed: ${audit.criticals.length} criticals`);
      // Trigger your own notification, Slack message, etc.
      break;
    case 'pipeline-error':
      console.error('Pipeline error:', req.body.error);
      break;
  }

  res.sendStatus(200);
});
```

## Language Rules by Genre

Each genre enforces rewrite patterns. Examples:

```typescript
// xuanhuan — no raw numbers for power levels
// ✗ "火元从12缕增加到24缕"
// ✓ "手臂比先前有力了，握拳时指骨发紧"

// urban — no analytical language
// ✗ "迅速分析了当前的债务状况"
// ✓ "把那叠皱巴巴的白条翻了三遍"

// horror — no direct emotion statements
// ✗ "感到一阵恐惧"
// ✓ "后颈的汗毛一根根立起来"
```

## Daemon / Scheduler

```bash
inkos daemon start                        # Start background process
inkos daemon start --books 吞天魔帝,书名2  # Specific books only
inkos daemon status
inkos daemon logs --tail 50
inkos daemon stop
```

Scheduler behavior:
- Default 15-minute cycle between chapter attempts
- Processes multiple books in parallel (configurable)
- Auto-retries with higher temperature on audit failure
- Suspends a book after N consecutive failures
- Respects daily chapter limit per book

## Troubleshooting

**Audit scores fluctuate wildly (0–6 criticals same chapter)**  
Lock audit temperature: `inkos audit 书名 --chapter N --temp 0`  
In config: `"auditTemperature": 0`

**Revision introduces more AI markers than original**  
This is auto-detected and the revision is discarded. Ensure you're on v0.4+. Use `spot-fix` mode — `rewrite` mode is known to inflate AI markers ~6x.

**Spinoff references parent canon events incorrectly**  
Re-import the canon: `inkos import canon 番外书名 --from 正传书名 --force`  
Check `story/parent_canon.md` for the event timeline boundaries.

**Post-write validator keeps blocking on 了-sentences**  
Rule: no more than 3 consecutive sentences containing 了. Adjust in `book_rules.md`:  
```markdown
## validatorOverrides
consecutiveLE: 5
```

**Style guide not being applied**  
Run `inkos style import reference.txt 书名` and confirm `style_guide.md` exists in book root. Check `inkos book status 书名` shows `styleGuide: active`.

**AIGC detection API 401 errors**  
API keys for GPTZero/Originality go in env, not config:  
```bash
export GPTZERO_API_KEY=your_key
export ORIGINALITY_API_KEY=your_key
```
