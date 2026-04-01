```markdown
---
name: awesome-design-md
description: Use curated DESIGN.md files from popular websites to give AI coding agents instant design system context for building pixel-perfect UI.
triggers:
  - use a design system from a real website
  - add a DESIGN.md to my project
  - make my UI look like Stripe/Vercel/Linear
  - give my agent design context
  - apply a design system to my project
  - build UI matching a popular website style
  - use awesome-design-md design tokens
  - drop in a DESIGN.md file for my agent
---

# Awesome Design MD

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated collection of `DESIGN.md` files reverse-engineered from 55+ popular developer-focused websites. Drop one into your project root and any AI coding agent instantly understands your target visual style — colors, typography, components, spacing, and more — without Figma exports or JSON schemas.

---

## What DESIGN.md Does

`DESIGN.md` is a plain-text design system document (introduced by [Google Stitch](https://stitch.withgoogle.com/docs/design-md/overview/)) that AI agents read to generate consistent UI. It is the design equivalent of `AGENTS.md`:

| File | Who reads it | What it defines |
|------|-------------|-----------------|
| `AGENTS.md` | Coding agents | How to build the project |
| `DESIGN.md` | Design agents | How the project should look and feel |

Markdown is the format LLMs read best — no parsing, no tooling, no configuration required.

---

## Installation

### Option 1: Clone the full repo

```bash
git clone https://github.com/VoltAgent/awesome-design-md.git
```

### Option 2: Download a single DESIGN.md (curl)

```bash
# Example: Vercel design system
curl -O https://raw.githubusercontent.com/VoltAgent/awesome-design-md/main/design-md/vercel/DESIGN.md

# Example: Stripe design system
curl -O https://raw.githubusercontent.com/VoltAgent/awesome-design-md/main/design-md/stripe/DESIGN.md

# Example: Linear design system
curl -O https://raw.githubusercontent.com/VoltAgent/awesome-design-md/main/design-md/linear.app/DESIGN.md
```

### Option 3: Copy via GitHub UI

1. Browse to the site folder: `https://github.com/VoltAgent/awesome-design-md/tree/main/design-md/<site>/`
2. Open `DESIGN.md`
3. Click **Raw**, then save as `DESIGN.md` in your project root

---

## Repository Structure

```
awesome-design-md/
└── design-md/
    └── <site-name>/
        ├── DESIGN.md          # The design system (what agents read)
        ├── preview.html       # Visual catalog — light surfaces
        └── preview-dark.html  # Visual catalog — dark surfaces
```

---

## Available Design Systems (55 total)

### AI & Machine Learning
| Site | Style |
|------|-------|
| `claude` | Warm terracotta accent, clean editorial |
| `elevenlabs` | Dark cinematic, audio-waveform aesthetic |
| `mistral.ai` | French minimalism, purple-toned |
| `ollama` | Terminal-first, monochrome simplicity |
| `replicate` | Clean white canvas, code-forward |
| `runwayml` | Cinematic dark, media-rich layout |
| `voltagent` | Void-black canvas, emerald accent |
| `x.ai` | Stark monochrome, futuristic minimalism |

### Developer Tools
| Site | Style |
|------|-------|
| `cursor` | Sleek dark, gradient accents |
| `linear.app` | Ultra-minimal, precise, purple accent |
| `mintlify` | Clean, green-accented, reading-optimized |
| `posthog` | Playful dark UI, developer-friendly |
| `raycast` | Sleek dark chrome, vibrant gradients |
| `resend` | Minimal dark, monospace accents |
| `supabase` | Dark emerald theme, code-first |
| `vercel` | Black and white precision, Geist font |

### Enterprise & Consumer
| Site | Style |
|------|-------|
| `apple` | Premium white space, SF Pro, cinematic |
| `airbnb` | Warm coral, photography-driven, rounded |
| `spotify` | Vibrant green on dark, bold type |
| `stripe` | Signature purple gradients, weight-300 |
| `notion` | Warm minimalism, serif headings |
| `figma` | Vibrant multi-color, playful professional |

> See [full collection](https://github.com/VoltAgent/awesome-design-md#collection) for all 55 sites.

---

## What's Inside Each DESIGN.md

Every file follows the [Stitch DESIGN.md format](https://stitch.withgoogle.com/docs/design-md/format/) with extended sections:

| # | Section | What it captures |
|---|---------|-----------------|
| 1 | Visual Theme & Atmosphere | Mood, density, design philosophy |
| 2 | Color Palette & Roles | Semantic name + hex + functional role |
| 3 | Typography Rules | Font families, full hierarchy table |
| 4 | Component Stylings | Buttons, cards, inputs, nav with states |
| 5 | Layout Principles | Spacing scale, grid, whitespace |
| 6 | Depth & Elevation | Shadow system, surface hierarchy |
| 7 | Do's and Don'ts | Design guardrails, anti-patterns |
| 8 | Responsive Behavior | Breakpoints, touch targets |
| 9 | Agent Prompt Guide | Quick color reference, ready prompts |

---

## Usage with AI Coding Agents

### Step 1: Copy DESIGN.md to your project root

```bash
# Pick a style that matches your target aesthetic
cp awesome-design-md/design-md/vercel/DESIGN.md ./DESIGN.md
```

### Step 2: Tell your agent to use it

**Claude Code:**
```
Build me a landing page hero section. Follow the DESIGN.md in the project root for all styling decisions.
```

**Cursor:**
```
Create a dashboard layout. Use DESIGN.md for colors, typography, and component patterns.
```

**GitHub Copilot / Codex:**
```
Implement a pricing table component. Refer to DESIGN.md for the design system.
```

**Google Stitch:**
DESIGN.md is natively supported — Stitch reads it automatically when present in the project.

---

## Code Examples

### Example: Building a button using Vercel DESIGN.md

After copying `design-md/vercel/DESIGN.md` to your project:

```html
<!-- The agent will infer these styles from DESIGN.md -->
<button class="btn-primary">Deploy</button>

<style>
  /* Agent generates this from Vercel DESIGN.md tokens */
  .btn-primary {
    background-color: #000000;       /* --color-foreground */
    color: #ffffff;                  /* --color-background */
    font-family: 'Geist', sans-serif;
    font-size: 14px;
    font-weight: 500;
    padding: 8px 16px;
    border-radius: 6px;
    border: none;
    cursor: pointer;
    transition: opacity 0.2s ease;
  }
  .btn-primary:hover {
    opacity: 0.85;
  }
</style>
```

### Example: Stripe-style card component

```html
<!-- After: cp design-md/stripe/DESIGN.md ./DESIGN.md -->
<!-- Prompt: "Create a pricing card. Use DESIGN.md." -->

<div class="pricing-card">
  <div class="pricing-card__badge">Most Popular</div>
  <h3 class="pricing-card__title">Pro</h3>
  <div class="pricing-card__price">
    <span class="pricing-card__amount">$12</span>
    <span class="pricing-card__period">/month</span>
  </div>
  <button class="pricing-card__cta">Get started</button>
</div>

<style>
  /* Stripe DESIGN.md: purple gradients, weight-300 elegance */
  .pricing-card {
    background: linear-gradient(135deg, #6772e5 0%, #9b59b6 100%);
    border-radius: 12px;
    padding: 32px;
    color: #ffffff;
    font-family: 'Sohne', -apple-system, sans-serif;
    font-weight: 300;
    max-width: 320px;
  }
  .pricing-card__title {
    font-size: 24px;
    font-weight: 500;
    margin: 0 0 16px;
  }
  .pricing-card__amount {
    font-size: 48px;
    font-weight: 300;
  }
  .pricing-card__cta {
    background: rgba(255,255,255,0.2);
    border: 1px solid rgba(255,255,255,0.4);
    color: #ffffff;
    border-radius: 6px;
    padding: 12px 24px;
    font-weight: 500;
    cursor: pointer;
    width: 100%;
    margin-top: 24px;
    backdrop-filter: blur(4px);
  }
</style>
```

### Example: Supabase-style dark dashboard layout

```html
<!-- After: cp design-md/supabase/DESIGN.md ./DESIGN.md -->
<!-- Prompt: "Build a sidebar nav layout. Follow DESIGN.md." -->

<!DOCTYPE html>
<html>
<head>
  <style>
    /* Supabase DESIGN.md: dark emerald theme, code-first */
    :root {
      --bg-primary: #1c1c1c;
      --bg-secondary: #242424;
      --bg-surface: #2a2a2a;
      --accent: #3ecf8e;
      --text-primary: #ededed;
      --text-muted: #8a8a8a;
      --border: #333333;
    }
    body {
      background: var(--bg-primary);
      color: var(--text-primary);
      font-family: 'Inter', sans-serif;
      margin: 0;
      display: flex;
      min-height: 100vh;
    }
    .sidebar {
      width: 240px;
      background: var(--bg-secondary);
      border-right: 1px solid var(--border);
      padding: 16px 0;
    }
    .sidebar-item {
      padding: 8px 16px;
      font-size: 13px;
      color: var(--text-muted);
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 8px;
    }
    .sidebar-item.active {
      color: var(--accent);
      background: rgba(62, 207, 142, 0.08);
    }
    .main-content {
      flex: 1;
      padding: 32px;
    }
    .stat-card {
      background: var(--bg-surface);
      border: 1px solid var(--border);
      border-radius: 8px;
      padding: 20px;
      display: inline-block;
      min-width: 180px;
    }
    .stat-value {
      font-size: 28px;
      font-weight: 600;
      color: var(--accent);
    }
    .stat-label {
      font-size: 12px;
      color: var(--text-muted);
      margin-top: 4px;
    }
  </style>
</head>
<body>
  <nav class="sidebar">
    <div class="sidebar-item active">📊 Dashboard</div>
    <div class="sidebar-item">🗄️ Database</div>
    <div class="sidebar-item">🔐 Auth</div>
    <div class="sidebar-item">📦 Storage</div>
    <div class="sidebar-item">⚡ Edge Functions</div>
  </nav>
  <main class="main-content">
    <h1>Project Overview</h1>
    <div style="display:flex;gap:16px;flex-wrap:wrap;margin-top:24px;">
      <div class="stat-card">
        <div class="stat-value">2.4M</div>
        <div class="stat-label">Database Rows</div>
      </div>
      <div class="stat-card">
        <div class="stat-value">12.8k</div>
        <div class="stat-label">Active Users</div>
      </div>
      <div class="stat-card">
        <div class="stat-value">99.9%</div>
        <div class="stat-label">Uptime</div>
      </div>
    </div>
  </main>
</body>
</html>
```

### Example: Programmatically selecting a DESIGN.md

```bash
#!/bin/bash
# select-design.sh — pick a design system by category

REPO="https://raw.githubusercontent.com/VoltAgent/awesome-design-md/main/design-md"

select_design() {
  local site=$1
  echo "Downloading DESIGN.md for: $site"
  curl -fsSL "$REPO/$site/DESIGN.md" -o DESIGN.md
  echo "✓ DESIGN.md ready. Tell your agent: 'Use DESIGN.md for all styling.'"
}

# Usage examples
case "$1" in
  vercel)   select_design "vercel" ;;
  stripe)   select_design "stripe" ;;
  linear)   select_design "linear.app" ;;
  supabase) select_design "supabase" ;;
  notion)   select_design "notion" ;;
  apple)    select_design "apple" ;;
  spotify)  select_design "spotify" ;;
  *)
    echo "Available: vercel, stripe, linear, supabase, notion, apple, spotify"
    echo "Usage: ./select-design.sh <site>"
    ;;
esac
```

---

## Viewing Previews Locally

Each design system ships with an HTML preview showing the actual color swatches, type scale, buttons, and cards:

```bash
# Open light preview
open design-md/stripe/preview.html

# Open dark preview
open design-md/stripe/preview-dark.html

# Or serve locally
cd design-md/vercel && python3 -m http.server 8080
# Visit http://localhost:8080/preview.html
```

---

## Agent Prompt Patterns

These prompts work reliably across Claude Code, Cursor, and Codex when `DESIGN.md` is in the project root:

### Landing page
```
Build a SaaS landing page hero with headline, subheadline, CTA button, and 
feature grid. Follow DESIGN.md strictly for all colors, fonts, spacing, 
and component styles.
```

### Dashboard
```
Create a responsive dashboard with sidebar navigation, stat cards, and a 
data table. Use the design tokens and component patterns from DESIGN.md.
```

### Component library
```
Generate a component file with: primary button, secondary button, input field, 
card, and badge. All styles must match the DESIGN.md specification exactly.
```

### Dark mode page
```
Build a dark-mode marketing page for a developer tool. Apply the dark surface 
colors, accent palette, and typography hierarchy defined in DESIGN.md.
```

### Specific component
```
Create a pricing table with 3 tiers. Use the card component styles, color 
roles, and button variants from DESIGN.md. The middle tier should use the 
primary accent color as the highlight.
```

---

## Choosing the Right DESIGN.md

| Your goal | Recommended site |
|-----------|-----------------|
| Clean developer tool / SaaS | `vercel`, `linear.app`, `resend` |
| Dark developer dashboard | `supabase`, `posthog`, `cursor` |
| Premium / luxury product | `apple`, `stripe`, `bmw` |
| Playful / friendly product | `figma`, `lovable`, `zapier` |
| AI / ML product | `voltagent`, `replicate`, `mistral.ai` |
| E-commerce / marketplace | `airbnb`, `pinterest` |
| Fintech / crypto | `stripe`, `revolut`, `coinbase` |
| Documentation site | `mintlify`, `hashicorp` |
| Media / content | `spotify`, `runwayml` |
| Enterprise SaaS | `ibm`, `hashicorp`, `sentry` |

---

## Contributing a New DESIGN.md

```bash
git clone https://github.com/VoltAgent/awesome-design-md.git
cd awesome-design-md

# Create folder for the new site
mkdir -p design-md/newsite.com

# Follow the format spec:
# https://stitch.withgoogle.com/docs/design-md/format/
# Include all 9 sections listed in the README

# Add your files
touch design-md/newsite.com/DESIGN.md
touch design-md/newsite.com/preview.html
touch design-md/newsite.com/preview-dark.html

# Submit PR
git checkout -b add-newsite-design
git add design-md/newsite.com/
git commit -m "feat: add newsite.com design system"
git push origin add-newsite-design
```

---

## Troubleshooting

### Agent ignores DESIGN.md
- Ensure the file is named exactly `DESIGN.md` (uppercase) in the **project root**
- Explicitly reference it in your prompt: *"Refer to the DESIGN.md file in the project root"*
- Some agents need the file added to context manually — drag it into the chat window

### Styles look generic / off
- The agent may be hallucinating styles. Add: *"Do not use any colors, fonts, or spacing values not defined in DESIGN.md"*
- Check the `preview.html` to verify you have the right design system
- Try a more specific prompt referencing exact section names, e.g. *"Use the Color Palette section of DESIGN.md"*

### Wrong site's DESIGN.md
- Run `head -5 DESIGN.md` to confirm which site's system is loaded
- Each DESIGN.md starts with the site name and theme description

### Preview HTML won't open
- These are static files — open directly in browser or serve with `python3 -m http.server`
- No build step or dependencies required

### Design tokens don't match the live site
- These are reverse-engineered approximations, not official design systems
- For official tokens, check each company's public design system documentation
- Open an issue at [github.com/VoltAgent/awesome-design-md/issues](https://github.com/VoltAgent/awesome-design-md/issues)

---

## Links

- **Repo:** https://github.com/VoltAgent/awesome-design-md
- **DESIGN.md Format Spec:** https://stitch.withgoogle.com/docs/design-md/format/
- **Google Stitch Overview:** https://stitch.withgoogle.com/docs/design-md/overview/
- **VoltAgent Framework:** https://github.com/VoltAgent/voltagent
- **Discord:** https://s.voltagent.dev/discord
- **License:** MIT
```
