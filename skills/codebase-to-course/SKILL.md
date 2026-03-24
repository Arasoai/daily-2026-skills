```markdown
---
name: codebase-to-course
description: Generate beautiful, interactive single-page HTML courses from any codebase using Claude Code — teaching non-technical vibe coders how their code works through animations, quizzes, and plain-English translations.
triggers:
  - "turn this codebase into a course"
  - "explain this codebase interactively"
  - "make a course from this project"
  - "teach me how this code works"
  - "interactive tutorial from this code"
  - "generate a course from this repo"
  - "create an HTML course for this project"
  - "make this codebase learnable"
---

# Codebase to Course

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A Claude Code skill that turns any codebase into a beautiful, self-contained single-page HTML course. Designed for "vibe coders" — people who build with AI tools and want to understand their own codebase well enough to steer AI better, debug more effectively, and talk to engineers confidently.

## What it produces

A **single HTML file** with zero dependencies that works offline, containing:

- Scroll-based modules with progress tracking and keyboard navigation
- Code ↔ Plain English side-by-side translations using real code from the repo
- Animated visualizations — data flow, component group chats, architecture diagrams
- Interactive quizzes testing *application* ("which files change if you add favorites?"), not memorization
- Glossary tooltips on hover for every technical term
- Warm, distinctive design (not the typical purple-gradient AI aesthetic)

## Installation

### As a Claude Code skill

```bash
# Clone the repo
git clone https://github.com/zarazhangrui/codebase-to-course

# Install the skill
cp -r codebase-to-course/codebase-to-course ~/.claude/skills/
```

The skill folder structure after install:

```
~/.claude/skills/codebase-to-course/
├── SKILL.md                    # Main skill instructions Claude reads
└── references/
    ├── design-system.md        # CSS tokens, typography, colors, layout rules
    └── interactive-elements.md # Quiz, animation, visualization code patterns
```

### Verify installation

Open any project in Claude Code and say one of the trigger phrases. Claude will read `SKILL.md` and the reference files before generating the course.

## How to use

### Basic invocation

Open a project in Claude Code and say any of:

```
"Turn this codebase into an interactive course"
"Teach me how this code works"
"Make a course from this project"
"Explain this codebase interactively"
```

Claude will:
1. Analyze the codebase structure (entry points, key files, data flow)
2. Identify the most important concepts for a vibe coder to understand
3. Generate a single `course.html` file in the project root

### What Claude analyzes

Claude reads and synthesizes:
- Entry points (`main.py`, `index.js`, `app.py`, `server.ts`, etc.)
- Key architectural files (routes, models, components, services)
- Config files to understand environment and dependencies
- README and any existing docs for intent/context

### Output location

By default, the course is saved as `course.html` in the project root. You can specify a different location:

```
"Turn this into a course and save it as docs/tutorial.html"
```

## Course structure Claude generates

Each course follows this module pattern:

```
Module 1: What Does This App Actually Do?
  → The "user journey" — what happens when someone uses it
  → Architecture overview diagram (animated)

Module 2: The Pieces (Components/Files/Services)
  → What each major piece does in plain English
  → How they talk to each other (group chat visualization)

Module 3: The Data
  → What gets stored, where, in what shape
  → Data flow animation (user action → processing → storage → response)

Module 4: [Feature-specific modules]
  → Real code snippets with side-by-side plain English
  → Quizzes on applying the concept

Module 5: Now You Can...
  → What you can now do better (steer AI, debug, talk to engineers)
  → Glossary of all technical terms used
```

## Design system (from `references/design-system.md`)

Claude follows strict design rules when generating courses:

```css
/* Core design tokens Claude uses */
:root {
  --bg: #faf9f7;           /* Warm off-white background */
  --text: #1a1916;         /* Near-black text */
  --accent: #d4651f;       /* Warm orange accent */
  --accent-light: #f5e6d8; /* Light accent for highlights */
  --mono: 'JetBrains Mono', monospace;
  --sans: 'Inter', system-ui, sans-serif;
}
```

Key layout rules:
- Every screen is at least 50% visual (diagram, animation, or interactive element)
- Max 2–3 sentences per text block
- Code snippets are **exact copies** from the real codebase — never simplified or modified
- No recycled metaphors — each concept gets a fresh one

## Interactive elements (from `references/interactive-elements.md`)

### Quiz pattern

```html
<!-- Claude generates quizzes like this — testing application, not memory -->
<div class="quiz-block" data-quiz-id="unique-id">
  <p class="quiz-question">
    A user reports stale data after switching pages. Where would you look first?
  </p>
  <div class="quiz-options">
    <button class="quiz-option" data-correct="false">
      The database schema
    </button>
    <button class="quiz-option" data-correct="true">
      The cache invalidation logic in <code>src/hooks/useData.ts</code>
    </button>
    <button class="quiz-option" data-correct="false">
      The CSS for the loading spinner
    </button>
  </div>
  <div class="quiz-feedback quiz-feedback--hidden">
    <p class="quiz-explanation">
      Stale data across page switches usually means the cache isn't being 
      cleared when it should be. The <code>useData</code> hook controls 
      when data gets refreshed.
    </p>
  </div>
</div>
```

### Data flow animation pattern

```html
<!-- Claude generates SVG animations showing data moving through the system -->
<div class="flow-animation" data-flow="user-request">
  <svg viewBox="0 0 800 200">
    <!-- Nodes: Browser → API → Auth → DB → Response -->
    <g class="flow-node" data-step="1">
      <rect x="20" y="80" width="120" height="40" rx="8"/>
      <text x="80" y="104">Browser</text>
    </g>
    <!-- Animated path between nodes -->
    <path class="flow-path" d="M 140 100 L 220 100" 
          stroke-dasharray="80" stroke-dashoffset="80">
      <animate attributeName="stroke-dashoffset" 
               from="80" to="0" dur="0.5s" 
               begin="flow-step-1.end"/>
    </path>
    <!-- ... more nodes ... -->
  </svg>
  <div class="flow-caption">
    <span class="flow-step-label" data-step="1">
      You click "Submit" — your browser packages the form data
    </span>
  </div>
</div>
```

### Glossary tooltip pattern

```html
<!-- Technical terms get automatic tooltips -->
<span class="glossary-term" 
      data-definition="A function that runs when data changes — 
                        like a notification that says 'hey, update yourself'">
  useEffect
</span>
```

### Code translation block pattern

```html
<!-- Real code on the left, plain English on the right -->
<div class="translation-block">
  <div class="translation-code">
    <pre><code class="language-typescript">
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const id = searchParams.get('id')
  const item = await db.items.findUnique({ where: { id } })
  if (!item) return NextResponse.json({ error: 'Not found' }, { status: 404 })
  return NextResponse.json(item)
}
    </code></pre>
    <p class="code-source">From: <code>app/api/items/route.ts</code></p>
  </div>
  <div class="translation-english">
    <p>
      When someone asks for a specific item by ID, look it up in the database. 
      If it doesn't exist, say "not found" (404). If it does, send it back.
    </p>
    <p>
      This is the <strong>API endpoint</strong> — the door your frontend 
      knocks on to get data.
    </p>
  </div>
</div>
```

## Directing Claude for better courses

### Specify your goal

```
"Make a course from this project — I want to understand it well enough 
to add a user favorites feature with AI help"
```

```
"Turn this into a course focused on the authentication flow — 
I keep breaking login when I add features"
```

### Specify what to skip

```
"Make a course from this, but skip the payment processing parts — 
focus on the core CRUD operations"
```

### Specify your background

```
"Make this into a course assuming I know what an API is but have 
never touched a database before"
```

### Ask for a specific module

```
"Just generate the data flow module for this codebase — 
I already understand the architecture"
```

## Common patterns Claude follows

### Metaphor rules

Each concept gets a unique metaphor that fits it specifically:

| Concept | Metaphor Claude might use |
|---|---|
| Database | Library with a card catalog |
| Auth/JWT | Wristband from a bouncer — proves you paid to get in |
| API | A restaurant menu — you can only order what's listed |
| Cache | Sticky note on your monitor vs. going to the filing cabinet |
| Webhook | A friend who texts you when something happens vs. you calling to check |
| Environment variables | The settings hidden behind a panel — not in the blueprint |

### What makes a good quiz question (Claude generates these)

❌ Bad: "What does API stand for?"  
✅ Good: "You want to add a 'favorites' feature. Based on what you learned, which files would you need to change?"

❌ Bad: "What HTTP method does GET use?"  
✅ Good: "A user says their profile changes aren't saving. You check the network tab and see a 200 OK response. Where would you look next?"

## Troubleshooting

### Course is too generic

**Problem:** Claude generated a course that could apply to any project.  
**Fix:** Tell Claude to read specific files first:

```
"Before making the course, read src/app/page.tsx, lib/db.ts, and 
api/routes/index.ts — the course should be specific to how THIS app works"
```

### Course is too technical

**Problem:** Explanations still feel like documentation.  
**Fix:** Redirect Claude's tone:

```
"Regenerate the course — explain everything as if you're texting a 
smart friend who has never coded but uses apps all day"
```

### Course is missing a key concept

**Problem:** Something important to the codebase isn't covered.  
**Fix:** Be explicit:

```
"The course is missing how the real-time updates work (Socket.io). 
Add a module between Module 3 and 4 that explains that with an animation"
```

### HTML file is too large to open smoothly

**Problem:** Very large codebases produce a very large HTML file.  
**Fix:** Scope the course:

```
"Make a focused course — just the authentication system and the main 
data models. Skip third-party integrations."
```

### Quizzes are too easy

**Problem:** Quiz questions are testing recall not application.  
**Fix:**

```
"The quizzes feel like trivia. Regenerate them so every question 
starts with a scenario: 'A user reports that...' or 'You want to add...'"
```

## Design principles to reinforce

If Claude drifts from the design philosophy, remind it:

```
"Remember: every screen should be at least 50% visual. 
If you wrote three paragraphs in a row, turn one into a diagram."
```

```
"Code snippets must be exact copies from the actual files — 
don't simplify or modify the real code."
```

```
"Quizzes test doing, not knowing. 
No definition questions — only scenario questions."
```

## Project links

- **Repository:** https://github.com/zarazhangrui/codebase-to-course
- **Built by:** [Zara Zhang Rui](https://x.com/zarazhangrui)
- **Skill install path:** `~/.claude/skills/codebase-to-course/`
```
