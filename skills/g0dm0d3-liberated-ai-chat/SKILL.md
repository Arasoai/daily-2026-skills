```markdown
---
name: g0dm0d3-liberated-ai-chat
description: Expert skill for G0DM0D3, a single-file multi-model AI chat interface with GODMODE, ULTRAPLINIAN, Parseltongue, AutoTune, and STM modules via OpenRouter
triggers:
  - set up G0DM0D3 chat interface
  - configure GODMODE or ULTRAPLINIAN mode
  - deploy liberated AI chat
  - use Parseltongue red teaming
  - integrate G0DM0D3 API with OpenRouter
  - add AutoTune adaptive sampling
  - self-host G0DM0D3 single file app
  - use G0DM0D3 multi-model evaluation
---

# G0DM0D3 Liberated AI Chat

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

G0DM0D3 is a single-file (`index.html`) open-source, privacy-first, multi-model AI chat interface built for red teaming, cognition research, and liberated AI interaction. It routes to 55+ models via OpenRouter and includes specialized engines: GODMODE CLASSIC (5 parallel model/prompt combos), ULTRAPLINIAN (up to 51-model comparative evaluation), Parseltongue (input perturbation for red teaming), AutoTune (adaptive sampling parameters), and STM Modules (semantic output normalization).

---

## Installation & Setup

### Option 1: Direct File (No Build Step)

```bash
git clone https://github.com/elder-plinius/G0DM0D3.git
cd G0DM0D3
open index.html
# or serve locally:
python3 -m http.server 8000
```

### Option 2: Static Hosting (GitHub Pages, Vercel, Netlify, Cloudflare Pages)

Upload `index.html` as the root static asset. No build process, no dependencies.

### Option 3: Docker (API Server)

```bash
cd api/
docker build -t g0dm0d3-api .
docker run -p 3000:3000 \
  -e OPENROUTER_API_KEY=$OPENROUTER_API_KEY \
  g0dm0d3-api
```

### API Key Configuration

G0DM0D3 never sends your API key to its own servers. The key is stored in browser `localStorage` only.

In the UI: **Settings → API Key → Enter your OpenRouter key**

Programmatically (for testing or embedding):

```javascript
localStorage.setItem('openrouter_api_key', process.env.OPENROUTER_API_KEY);
```

Get an OpenRouter key at: https://openrouter.ai/keys

---

## Architecture Overview

```
G0DM0D3/
├── index.html        # Entire application: UI + logic + styles (vanilla JS)
├── api/              # Optional Node.js/Express API server
│   ├── server.js     # Express server wrapping OpenRouter
│   └── Dockerfile
├── API.md            # REST API reference
├── PAPER.md          # Research paper on modules
├── TERMS.md          # Privacy & data policy
└── SECURITY.md       # Vulnerability reporting
```

All core logic lives in `index.html` as vanilla HTML/CSS/JavaScript — no framework, no bundler.

---

## Core Modes

### GODMODE CLASSIC

Fires 5 model+prompt combos in parallel. Each combo uses a different battle-tested system prompt strategy. The best response is surfaced.

| Combo | Model ID | Strategy |
|-------|----------|----------|
| 🩷 CLAUDE 3.5 SONNET | `anthropic/claude-3.5-sonnet` | END/START boundary inversion |
| 💜 GROK 3 | `x-ai/grok-3` | Unfiltered liberated + GODMODE divider |
| 💙 GEMINI 2.5 FLASH | `google/gemini-2.5-flash` | Refusal inversion + rebel genius |
| 💛 GPT-4 CLASSIC | `openai/gpt-4o` | OG GODMODE l33t format |
| 💚 GODMODE FAST | `nousresearch/hermes-4-405b` | Instant stream, zero refusal checking |

### ULTRAPLINIAN Tiers

| Tier | Models | Use Case |
|------|--------|----------|
| ⚡ FAST | 10 | Quick comparisons |
| 🎯 STANDARD | 24 | Balanced evaluation |
| 🧠 SMART | 36 | Reasoning-heavy tasks |
| ⚔️ POWER | 45 | Frontier model coverage |
| 🔱 ULTRA | 51 | Full model sweep |

---

## API Server Usage

The optional `api/` server exposes an OpenAI-compatible REST interface.

### Base URL

```
http://localhost:3000/v1
```

### Chat Completions (OpenAI-compatible)

```typescript
import OpenAI from 'openai';

const client = new OpenAI({
  baseURL: 'http://localhost:3000/v1',
  apiKey: process.env.OPENROUTER_API_KEY,
});

const response = await client.chat.completions.create({
  model: 'anthropic/claude-3.5-sonnet',
  messages: [{ role: 'user', content: 'Explain quantum entanglement.' }],
  stream: false,
});

console.log(response.choices[0].message.content);
```

### GODMODE CLASSIC via API

```typescript
const response = await fetch('http://localhost:3000/v1/godmode', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${process.env.OPENROUTER_API_KEY}`,
  },
  body: JSON.stringify({
    message: 'Your prompt here',
    mode: 'classic', // 'classic' | 'ultraplinian'
  }),
});

const data = await response.json();
console.log(data.winner);     // Best response
console.log(data.responses);  // All 5 responses
```

### ULTRAPLINIAN via API

```typescript
const response = await fetch('http://localhost:3000/v1/ultraplinian', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${process.env.OPENROUTER_API_KEY}`,
  },
  body: JSON.stringify({
    message: 'Compare approaches to AGI safety.',
    tier: 'SMART', // 'FAST' | 'STANDARD' | 'SMART' | 'POWER' | 'ULTRA'
  }),
});

const data = await response.json();
// data.winner: { model, response, score }
// data.scores: [{ model, score, breakdown }]
console.log(`Winner: ${data.winner.model} (score: ${data.winner.score})`);
```

### Streaming Chat

```typescript
const stream = await client.chat.completions.create({
  model: 'google/gemini-2.5-flash',
  messages: [{ role: 'user', content: 'Write a haiku about recursion.' }],
  stream: true,
});

for await (const chunk of stream) {
  const delta = chunk.choices[0]?.delta?.content ?? '';
  process.stdout.write(delta);
}
```

---

## Parseltongue (Input Perturbation Engine)

Used for red-teaming research. Detects trigger words and applies obfuscation techniques.

### Configuration in UI

Settings → Parseltongue → Intensity: `light` | `medium` | `heavy`

### Techniques Available

| Technique | Example |
|-----------|---------|
| Leetspeak | `hello` → `h3ll0` |
| Bubble text | `hello` → `ʰᵉˡˡᵒ` |
| Braille | Unicode braille substitution |
| Morse code | `hello` → `.... . .-.. .-.. ---` |
| Unicode substitution | Lookalike characters |
| Phonetic | `hello` → `hotel echo lima lima oscar` |

### Trigger Tiers

- **Light (11 triggers)**: Common flagged terms
- **Standard (22 triggers)**: Extended list
- **Heavy (33 triggers)**: Full trigger set

### Programmatic Parseltongue (within index.html context)

```javascript
// Access the Parseltongue engine from browser console or embedded script
const perturbed = window.parseltongue.perturb(inputText, {
  intensity: 'medium',   // 'light' | 'medium' | 'heavy'
  techniques: ['leetspeak', 'unicode'], // subset or all
});
console.log(perturbed);
```

---

## AutoTune (Adaptive Sampling)

Classifies queries into 5 context types and sets optimal sampling parameters automatically. Uses EMA (Exponential Moving Average) from thumbs up/down feedback.

### Context Types & Default Parameters

| Context | Temperature | Top-P | Top-K | Freq Penalty | Presence Penalty |
|---------|-------------|-------|-------|--------------|-----------------|
| Creative | 1.1 | 0.95 | 80 | 0.3 | 0.4 |
| Technical | 0.3 | 0.85 | 40 | 0.1 | 0.1 |
| Factual | 0.2 | 0.80 | 30 | 0.0 | 0.0 |
| Conversational | 0.7 | 0.90 | 50 | 0.2 | 0.2 |
| Analytical | 0.5 | 0.88 | 60 | 0.15 | 0.15 |

### Manual Override in UI

Settings → AutoTune → Toggle off to use manual sliders for temperature, top_p, etc.

### API-Level Parameter Control

```typescript
const response = await client.chat.completions.create({
  model: 'openai/gpt-4o',
  messages: [{ role: 'user', content: 'Write a surrealist short story.' }],
  temperature: 1.1,
  top_p: 0.95,
  frequency_penalty: 0.3,
  presence_penalty: 0.4,
});
```

---

## STM Modules (Semantic Transformation Modules)

Post-process AI output in real-time within the browser.

| Module | Effect |
|--------|--------|
| **Hedge Reducer** | Removes "I think", "maybe", "perhaps", "it seems" |
| **Direct Mode** | Strips preambles like "Certainly!", "Of course!", "Great question!" |
| **Curiosity Bias** | Appends exploration/follow-up prompts |

Enable in: **Settings → STM Modules → Toggle individual modules**

### Custom STM in JavaScript

```javascript
// Programmatic hedge reduction (mirrors internal logic)
function reduceHedges(text) {
  const hedges = [
    /\bI think\b/gi,
    /\bperhaps\b/gi,
    /\bmaybe\b/gi,
    /\bit seems\b/gi,
    /\bI believe\b/gi,
    /\bone might argue\b/gi,
  ];
  return hedges.reduce((t, re) => t.replace(re, ''), text).trim();
}

// Direct mode — strip filler preambles
function directMode(text) {
  return text.replace(
    /^(Certainly!|Of course!|Great question!|Sure!|Absolutely!)\s*/i,
    ''
  );
}
```

---

## Themes

| Theme | CSS Class | Aesthetic |
|-------|-----------|-----------|
| Matrix | `theme-matrix` | Green terminal |
| Hacker | `theme-hacker` | Red/orange cyberpunk |
| Glyph | `theme-glyph` | Purple mystical |
| Minimal | `theme-minimal` | Clean light mode |

Switch via: **Settings → Theme**

---

## Privacy & Data Handling

### Key Guarantees

- API key: `localStorage` only, never transmitted to G0DM0D3 servers
- No cookies, no PII collection, no login required
- Telemetry: structural/anonymous only, **opt-out** in Settings → Privacy
- Chat history: browser `localStorage` only — no cloud sync

### Chat History Management

```javascript
// Export chat history
const history = JSON.parse(localStorage.getItem('g0dm0d3_history') || '[]');
const blob = new Blob([JSON.stringify(history, null, 2)], { type: 'application/json' });
const url = URL.createObjectURL(blob);
// Trigger download...

// Import chat history
const imported = JSON.parse(await file.text());
localStorage.setItem('g0dm0d3_history', JSON.stringify(imported));
```

**Warning**: Clearing browser data permanently deletes all chat history. Use Settings → Data → Export regularly.

### Opt-Out Telemetry

```javascript
localStorage.setItem('g0dm0d3_telemetry_opt_out', 'true');
```

### Dataset Generation (Self-Hosted API Only)

The open research dataset feature (publishes to HuggingFace) is:
- **OFF by default**
- Only available when running the Docker API server
- NOT present on godmod3.ai
- Requires explicit consent modal

---

## Embedding G0DM0D3 in Your Project

Since it's a single HTML file, embed as an iframe:

```html
<iframe
  src="/path/to/index.html"
  width="100%"
  height="800px"
  style="border: none;"
  title="G0DM0D3 Chat"
></iframe>
```

Or load dynamically:

```typescript
async function loadG0DM0D3(containerId: string): Promise<void> {
  const response = await fetch('/g0dm0d3/index.html');
  const html = await response.text();
  const container = document.getElementById(containerId);
  if (!container) return;
  
  const iframe = document.createElement('iframe');
  iframe.srcdoc = html;
  iframe.style.cssText = 'width:100%;height:100%;border:none;';
  container.appendChild(iframe);
}
```

---

## OpenRouter Model IDs Reference

```typescript
const MODELS = {
  // Anthropic
  CLAUDE_SONNET: 'anthropic/claude-3.5-sonnet',
  CLAUDE_HAIKU: 'anthropic/claude-3.5-haiku',
  
  // OpenAI
  GPT4O: 'openai/gpt-4o',
  GPT4O_MINI: 'openai/gpt-4o-mini',
  
  // Google
  GEMINI_FLASH: 'google/gemini-2.5-flash',
  GEMINI_PRO: 'google/gemini-2.5-pro',
  
  // xAI
  GROK3: 'x-ai/grok-3',
  
  // Meta
  LLAMA_70B: 'meta-llama/llama-3.3-70b-instruct',
  
  // Mistral
  MISTRAL_LARGE: 'mistralai/mistral-large',
  
  // DeepSeek
  DEEPSEEK_R1: 'deepseek/deepseek-r1',
  
  // Nous Research
  HERMES: 'nousresearch/hermes-4-405b',
} as const;
```

---

## Common Patterns

### Pattern: Multi-Model Comparison Script

```typescript
import OpenAI from 'openai';

const client = new OpenAI({
  baseURL: 'http://localhost:3000/v1',
  apiKey: process.env.OPENROUTER_API_KEY,
});

const models = [
  'anthropic/claude-3.5-sonnet',
  'openai/gpt-4o',
  'google/gemini-2.5-flash',
  'x-ai/grok-3',
];

async function compareModels(prompt: string) {
  const results = await Promise.allSettled(
    models.map(async (model) => {
      const res = await client.chat.completions.create({
        model,
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 500,
      });
      return {
        model,
        response: res.choices[0].message.content,
        tokens: res.usage?.total_tokens,
      };
    })
  );

  return results
    .filter((r): r is PromiseFulfilledResult<any> => r.status === 'fulfilled')
    .map((r) => r.value);
}

const comparisons = await compareModels('What is consciousness?');
comparisons.forEach(({ model, response, tokens }) => {
  console.log(`\n--- ${model} (${tokens} tokens) ---`);
  console.log(response);
});
```

### Pattern: Streaming with Progress Tracking

```typescript
async function streamWithTracking(prompt: string, model: string) {
  const stream = await client.chat.completions.create({
    model,
    messages: [{ role: 'user', content: prompt }],
    stream: true,
    temperature: 0.7,
  });

  let fullResponse = '';
  let tokenCount = 0;

  for await (const chunk of stream) {
    const delta = chunk.choices[0]?.delta?.content ?? '';
    fullResponse += delta;
    tokenCount++;
    process.stdout.write(delta);

    if (chunk.choices[0]?.finish_reason === 'stop') {
      console.log(`\n\n[Complete — ~${tokenCount} chunks]`);
    }
  }

  return fullResponse;
}
```

### Pattern: AutoTune-Style Parameter Selection

```typescript
type ContextType = 'creative' | 'technical' | 'factual' | 'conversational' | 'analytical';

interface SamplingParams {
  temperature: number;
  top_p: number;
  frequency_penalty: number;
  presence_penalty: number;
}

const AUTOTUNE_PARAMS: Record<ContextType, SamplingParams> = {
  creative:       { temperature: 1.1, top_p: 0.95, frequency_penalty: 0.3, presence_penalty: 0.4 },
  technical:      { temperature: 0.3, top_p: 0.85, frequency_penalty: 0.1, presence_penalty: 0.1 },
  factual:        { temperature: 0.2, top_p: 0.80, frequency_penalty: 0.0, presence_penalty: 0.0 },
  conversational: { temperature: 0.7, top_p: 0.90, frequency_penalty: 0.2, presence_penalty: 0.2 },
  analytical:     { temperature: 0.5, top_p: 0.88, frequency_penalty: 0.15, presence_penalty: 0.15 },
};

function classifyContext(prompt: string): ContextType {
  const lower = prompt.toLowerCase();
  if (/write|story|poem|creative|imagine|fiction/.test(lower)) return 'creative';
  if (/code|function|implement|debug|syntax|error/.test(lower)) return 'technical';
  if (/what is|define|explain|fact|history|when/.test(lower)) return 'factual';
  if (/how are|feel|think|opinion|chat/.test(lower)) return 'conversational';
  return 'analytical';
}

async function autoTunedRequest(prompt: string, model: string) {
  const context = classifyContext(prompt);
  const params = AUTOTUNE_PARAMS[context];

  console.log(`[AutoTune] Context: ${context}`, params);

  return client.chat.completions.create({
    model,
    messages: [{ role: 'user', content: prompt }],
    ...params,
  });
}
```

---

## Troubleshooting

### API Key Not Working

- Ensure key is from https://openrouter.ai/keys
- Check the key has credits loaded
- Verify it's entered in Settings (stored in `localStorage`, not env at browser level)
- Test with: `curl -H "Authorization: Bearer $OPENROUTER_API_KEY" https://openrouter.ai/api/v1/models`

### CORS Errors (Self-Hosted API)

```javascript
// In api/server.js, ensure CORS is configured:
app.use(cors({
  origin: ['http://localhost:8000', 'https://yourdomain.com'],
  methods: ['GET', 'POST'],
}));
```

### Models Returning Empty Responses

Some models via OpenRouter have specific formatting requirements. Add `max_tokens`:

```typescript
const response = await client.chat.completions.create({
  model: 'deepseek/deepseek-r1',
  messages: [{ role: 'user', content: prompt }],
  max_tokens: 2048, // Explicit limit prevents silent truncation
});
```

### Chat History Lost

History lives in `localStorage` only. To preserve it:

1. Settings → Data → Export (downloads JSON)
2. On new device/browser: Settings → Data → Import

### index.html Rendering Blank

- Ensure you're opening via HTTP server, not `file://` protocol (some browsers block `file://` localStorage)
- Use `python3 -m http.server 8000` and open `http://localhost:8000`

### ULTRAPLINIAN Timeout

With 51 models in ULTRA tier, some requests may timeout. Switch to SMART (36) or STANDARD (24) for faster results. Add timeouts in API usage:

```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30000);

try {
  const response = await fetch('http://localhost:3000/v1/ultraplinian', {
    method: 'POST',
    signal: controller.signal,
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${process.env.OPENROUTER_API_KEY}` },
    body: JSON.stringify({ message: prompt, tier: 'SMART' }),
  });
  const data = await response.json();
  console.log(data.winner);
} finally {
  clearTimeout(timeout);
}
```

---

## Key Links

- **Hosted App**: https://godmod3.ai
- **OpenRouter Keys**: https://openrouter.ai/keys
- **OpenRouter Models**: https://openrouter.ai/models
- **API Docs**: [API.md](https://github.com/elder-plinius/G0DM0D3/blob/main/API.md)
- **Research Paper**: [PAPER.md](https://github.com/elder-plinius/G0DM0D3/blob/main/PAPER.md)
- **Terms & Privacy**: [TERMS.md](https://github.com/elder-plinius/G0DM0D3/blob/main/TERMS.md)
- **License**: AGPL-3.0
```
