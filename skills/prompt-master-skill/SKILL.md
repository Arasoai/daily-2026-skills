```markdown
---
name: prompt-master-skill
description: Claude skill that generates accurate, token-efficient prompts for any AI tool — Claude, GPT, Midjourney, Cursor, and 30+ more.
triggers:
  - write me a prompt for
  - generate a prompt for
  - help me prompt
  - fix my prompt
  - create a midjourney prompt
  - write a cursor prompt
  - make a prompt for claude code
  - I need a better prompt for
---

# Prompt Master

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Prompt Master is a Claude skill that writes sharp, token-efficient prompts for any AI tool. It eliminates the re-prompting loop by extracting full intent on the first pass, routing to the right prompt architecture, and auditing every word for necessity before delivery.

**Supports:** Claude, ChatGPT, Gemini, o1/o3/o4, DeepSeek, MiniMax, Qwen, Cursor, Windsurf, Claude Code, GitHub Copilot, Bolt, v0, Lovable, Devin, Perplexity, Midjourney, DALL-E, Stable Diffusion, ComfyUI, Sora, Runway, ElevenLabs, Zapier, Make, and any unknown tool via universal fingerprint.

---

## Installation

### Recommended — Claude.ai (browser)

1. Download the repo as a ZIP from [github.com/nidhinjs/prompt-master](https://github.com/nidhinjs/prompt-master)
2. Go to **claude.ai → Sidebar → Customize → Skills → Upload a Skill**

### Claude Code (CLI)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/nidhinjs/prompt-master.git ~/.claude/skills/prompt-master
```

Once installed, the skill activates automatically when Claude detects prompt-generation intent, or you can invoke it explicitly.

---

## How to Invoke

### Natural language (auto-detected)

```
Write me a prompt for Cursor to refactor my auth module
```

```
I need a prompt for Claude Code to build a REST API
```

```
Here's a bad prompt I wrote for GPT-4o, fix it: [paste prompt]
```

```
Generate a Midjourney prompt for a cyberpunk city at night
```

### Explicit invocation

```
/prompt-master

I want to ask Claude Code to build a todo app with React and Supabase
```

---

## What Happens Internally (The Pipeline)

Prompt Master runs a 7-step pipeline on every request. You never see the machinery — only the output.

| Step | What It Does |
|------|-------------|
| 1. **Tool Detection** | Identifies the target AI system and routes to its profile |
| 2. **Intent Extraction** | Pulls 9 dimensions: task, input, output, constraints, context, audience, memory, success criteria, examples |
| 3. **Clarifying Questions** | Asks ≤3 targeted questions if critical info is missing |
| 4. **Framework Routing** | Selects the right prompt architecture from 12 templates |
| 5. **Safe Techniques** | Applies role assignment, few-shot, XML structure, grounding anchors, memory blocks |
| 6. **Token Efficiency Audit** | Strips every word that doesn't change the output |
| 7. **Delivery** | One clean copyable block + one-line strategy note |

---

## Prompt Templates (Auto-Selected)

You never choose a template — Prompt Master picks it. Shown here for understanding:

| Template | Used For |
|----------|----------|
| **RTF** (Role, Task, Format) | Fast one-shot tasks |
| **CO-STAR** | Professional documents, business writing |
| **RISEN** | Complex multi-step projects |
| **CRISPE** | Creative work, brand voice, iterative content |
| **Chain of Thought** | Math, logic, debugging, multi-step analysis |
| **Few-Shot** | Consistent structured output, pattern replication |
| **File-Scope Template** | Cursor, Windsurf, Copilot — code editing AI |
| **ReAct + Stop Conditions** | Claude Code, Devin — autonomous agents |
| **Visual Descriptor** | Midjourney, Stable Diffusion, image AI |
| **Prose Description** | DALL-E 3 |
| **Node Split** | ComfyUI (positive/negative node separation) |
| **Universal Fingerprint** | Any unknown tool — 4 questions, then generates |

---

## Real Usage Examples

### Example 1 — Image prompt (Midjourney)

**You say:**
```
Write me a midjourney prompt for a realistic samurai standing in the rain at night
```

**Prompt Master delivers:**
```
lone samurai standing in heavy rain at night, traditional armor,
neon reflections on wet cobblestone street, cinematic lighting,
dramatic shadows, fog, ultra detailed, photorealistic,
shallow depth of field --ar 16:9 --v 6 --style raw

negative: blurry, low quality, watermark, cartoon, anime, extra limbs
```

> 🎯 Target: Midjourney · ⚡ Framework: Visual Descriptor · 💰 Tokens: Light (~60)
> 💡 Strategy: Comma-separated descriptors over prose, lighting anchored early, version locked, negative prompt prevents style drift.

---

### Example 2 — Coding prompt (Claude Code)

**You say:**
```
Build a claude code prompt for a landing page that looks like Notion
```

**Prompt Master delivers:**
```
Objective:
Build a fully detailed, production-quality static landing page for a
business dashboard product. Design must match Notion's exact aesthetic —
clean, minimal, typographic, with smooth animations throughout.

Stack:
Single HTML file with embedded CSS and vanilla JS. No frameworks.
No external dependencies except Google Fonts (Inter).

Design Spec — match these exactly:
- Background: #ffffff
- Primary text: #1a1a1a
- Secondary text: #6b7280
- Border color: #e5e7eb
- Accent: #000000
- Font: Inter, weights 400 / 500 / 600 / 700
- Base spacing unit: 8px
- Border radius: 6px on cards, 4px on buttons
- Box shadows: 0 1px 3px rgba(0,0,0,0.08) only

Sections to build in order:
1. Navbar — logo left, nav links center, CTA right. Sticky, border-bottom on scroll.
2. Hero — large left-aligned heading, subtext, two CTAs, mockup placeholder.
3. Logos bar — "Trusted by teams at..." with 5 placeholder names, muted gray.
4. Features — 3-column grid, icon + bold title + 2-line description.
   Hover: border darkens to #d1d5db, translateY(-2px), 200ms ease.
5. How it works — alternating left/right, 3 steps.
6. Pricing — 2 tiers. Free: outlined. Pro: black bg, white text, badge.
7. CTA banner — full-width dark section, centered heading, single CTA.
8. Footer — 4-column links, copyright.

Animations:
- Fade-in + translateY(20px→0) on scroll via IntersectionObserver. Threshold 0.15. 500ms ease-out.
- Feature cards: 100ms stagger delay.
- All hover transitions: 200ms ease.
- Smooth scroll on all anchor links.

Constraints:
- Single file only. All CSS and JS embedded.
- No dependencies except Google Fonts CDN.
- Placeholder images = styled gray divs.
- Pixel-perfect at 375px and 1440px.
- Zero console errors.

Done When:
All 8 sections render, animations trigger without layout shift,
hover states work, navbar sticks and border appears on scroll.
```

> 🎯 Target: Claude Code · 💰 Tokens: Medium (~380)
> 💡 Strategy: Every vague aesthetic cue translated into exact hex values and pixel specs — Claude Code cannot guess wrong.

---

### Example 3 — Fix a bad prompt

**You say:**
```
Here's a bad prompt I wrote for GPT-4o, fix it:

"write me some python code to handle user auth"
```

**Prompt Master delivers:**
```
You are a senior Python backend engineer.

Task: Implement a complete user authentication module.

Requirements:
- Framework: FastAPI
- Auth method: JWT (access token 15min, refresh token 7 days)
- Password hashing: bcrypt via passlib
- Endpoints: POST /auth/register, POST /auth/login, POST /auth/refresh, POST /auth/logout
- Store refresh tokens in Redis with TTL
- Return format: { access_token, refresh_token, token_type }

Constraints:
- No third-party auth services (Auth0, Firebase, etc.)
- No ORM — use raw asyncpg queries
- Include input validation with Pydantic v2 models
- Include error handling with proper HTTP status codes

Deliver:
- auth/router.py
- auth/schemas.py
- auth/utils.py
- auth/dependencies.py

Do not scaffold a full app. Auth module only.
```

> 🎯 Target: GPT-4o · ⚡ Framework: RTF + File-Scope
> 💡 Strategy: Vague "some python code" collapsed into exact stack, endpoints, constraints, and file deliverables.

---

### Example 4 — Stable Diffusion

**You say:**
```
Write a stable diffusion prompt for a futuristic city interior, warm lighting
```

**Prompt Master delivers:**
```
Positive:
futuristic city interior, (warm golden lighting:1.3), (volumetric light rays:1.2),
glass and steel architecture, holographic displays, busy atrium,
people walking, ultra detailed, 8k, cinematic composition,
(photorealistic:1.4), depth of field, bokeh background

Negative:
(blurry:1.3), (low quality:1.3), (watermark:1.2), cartoon, anime,
flat lighting, oversaturated, extra limbs, bad anatomy, deformed,
ugly, duplicate, morbid, mutilated, out of frame
```

> 🎯 Target: Stable Diffusion · ⚡ Framework: Node Split
> 💡 Strategy: Weight syntax applied to critical elements, mandatory negative prompt, photorealism locked.

---

### Example 5 — Workflow automation (Zapier)

**You say:**
```
Write a prompt for Zapier to automatically save Gmail attachments to Google Drive
```

**Prompt Master delivers:**
```
Trigger:
- App: Gmail
- Event: New Email Matching Search
- Search filter: has:attachment

Filter (add a Filter step):
- Only continue if: Attachment exists = True

Action 1:
- App: Gmail
- Action: Get Email
- Field: Use email ID from trigger step

Action 2:
- App: Google Drive
- Action: Upload File
- File: Attachment from Action 1
- Drive: My Drive
- Folder: /Gmail Attachments/{{trigger.from_email}}/{{trigger.date_formatted}}
- File name: {{trigger.subject}} - {{attachment.filename}}

Notes:
- Add a second Action 2 loop if email can have multiple attachments
- Set folder path dynamically using sender email and date to avoid clutter
- Test with a single known email before enabling live Zap
```

> 🎯 Target: Zapier · ⚡ Framework: Trigger-Action Map
> 💡 Strategy: Trigger app + event + filter + field mapping fully specified — no Zapier configuration guesswork.

---

## Tool Profile Reference

Prompt Master includes built-in profiles that know the quirks of each tool:

```
Claude        → XML structure, length spec, no padding
ChatGPT       → Output contract, verbosity control, completion criteria
Gemini        → Grounding anchors, citation rules, format locks
o3/o4-mini    → Short clean instructions only — never add CoT (they think internally)
DeepSeek-R1   → Short instructions, suppresses thinking output if needed
MiniMax       → Temperature hints, thinking tag control
Ollama        → Asks which model is loaded, includes system prompt for Modelfile
Cursor        → File path, function name, do-not-touch list
Claude Code   → Stop conditions, file scope, checkpoint output
Copilot       → Exact function contract as docstring
Bolt/v0       → Stack spec, version, what NOT to scaffold
Devin         → Starting state, target state, stop conditions
Midjourney    → Comma descriptors, --parameters, negative prompts
DALL-E 3      → Prose description, text exclusion, edit vs generate detection
Stable Diff   → Weight syntax (word:1.3), CFG, mandatory negatives
ComfyUI       → Positive/negative node split, checkpoint-specific syntax
Sora/Runway   → Camera movement, duration, cut style
ElevenLabs    → Emotion, pacing, emphasis, speech rate
Zapier/Make   → Trigger + event + action + field mapping
```

For any unlisted tool, Prompt Master uses the **Universal Fingerprint** — 4 questions to characterize the tool and generate a quality prompt anyway.

---

## Clarifying Questions Logic

Prompt Master asks ≤3 questions only when critical info is genuinely missing. It never asks for information it can reasonably infer.

**It will ask when:**
- Target tool is ambiguous and it changes the output significantly
- Task scope is completely undefined (e.g. "write a prompt for building an app" — what app?)
- Output format has high-stakes options (e.g. single file vs multi-file for Claude Code)

**It will NOT ask:**
- For style preferences it can default to sensibly
- For confirmation of things clearly stated
- More than 3 questions ever

---

## Key Principles

### The core insight
> "The best prompt is not the longest. It's the one where every word is load-bearing."

### What Prompt Master does differently from prompt generators
- **Other tools:** Make prompts longer
- **Prompt Master:** Makes prompts sharper — token efficiency audit strips everything that doesn't change the output

### The 9 dimensions of intent it extracts
1. Task — what action needs to happen
2. Input — what the AI is working with
3. Output — what the result should look like
4. Constraints — what must not happen
5. Context — background the AI needs
6. Audience — who the output is for
7. Memory — what should persist across turns
8. Success criteria — how to know it's done
9. Examples — patterns to replicate or avoid

---

## Troubleshooting

### Prompt Master isn't activating
- Ensure the skill is uploaded/installed correctly in Claude's skill directory
- Try explicit invocation: `/prompt-master [your request]`
- Make sure your request mentions a target tool or prompt-related intent

### Generated prompt is too long for my use case
- Tell Prompt Master the token budget: `"Write me a Midjourney prompt, keep it under 50 tokens"`
- It will re-run the token efficiency audit with a hard cap

### Generated prompt doesn't match my tool's syntax
- Name the exact tool version: `"for Stable Diffusion 1.5"` vs `"for SDXL"` — these have different optimal syntax
- If it's an obscure or new tool, paste 1-2 example prompts that work: `"Here's a prompt that worked for [Tool X], use that style"`

### It asked me questions but I want a prompt now
- Say: `"Just generate your best guess, I'll refine after"`
- Prompt Master will use reasonable defaults and flag its assumptions in the strategy note

### The prompt works but I want a variation
- Say: `"Give me 3 variations of that prompt, different approaches"`
- Or: `"Make it more cinematic"` / `"Make it more technical"` — it retains full context

### I want to see which framework it used
- Ask: `"Which prompt framework did you use and why?"`
- It will explain the routing decision

---

## Tips for Best Results

```
# Be specific about the target tool
"Write a prompt for Claude Code"        ✅
"Write a prompt for an AI"              ❌ (will ask for clarification)

# Include your stack when relevant
"Claude Code prompt for a React app using Supabase and Tailwind"   ✅
"Claude Code prompt for an app"                                     ❌

# Paste bad prompts for fixing — Prompt Master thrives on this
"Fix this: write me some code for auth"   ✅

# Give reference style when you have one
"Generate a Midjourney prompt like this example: [paste]"   ✅

# Specify constraints upfront
"Single file only, no external dependencies"   ✅ (saves a clarifying question round)
```

---

## Project Info

- **Repo:** [github.com/nidhinjs/prompt-master](https://github.com/nidhinjs/prompt-master)
- **License:** MIT
- **Topics:** claude-ai, claude-skills, llm, prompt-engineering
- **Works with:** Claude.ai browser skill system, Claude Code skill directory
```
