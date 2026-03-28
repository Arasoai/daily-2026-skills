```markdown
---
name: ai-website-cloner-template
description: Clone any website into a pixel-perfect Next.js replica using AI coding agents with one command
triggers:
  - clone this website
  - reverse engineer a website
  - scrape and rebuild a site
  - copy this website design
  - clone website with AI
  - rebuild this page as Next.js
  - extract design tokens from a site
  - run clone-website skill
---

# AI Website Cloner Template

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A multi-phase AI agent pipeline that reverse-engineers any website and rebuilds it as a pixel-perfect Next.js 16 clone — fonts, colors, assets, interactions, and responsive layout included. One command triggers reconnaissance, design extraction, parallel component building, and visual QA.

---

## Installation

```bash
git clone https://github.com/JCodesMore/ai-website-cloner-template.git my-clone
cd my-clone
npm install
```

Start your AI agent (Claude Code recommended):

```bash
claude --chrome
```

---

## Quick Start

```
/clone-website https://example.com
```

That's it. The skill handles everything: screenshots, token extraction, asset downloads, component specs, parallel builds, assembly, and visual diff.

---

## How the Pipeline Works

The `/clone-website` skill runs five sequential phases:

### Phase 1 — Reconnaissance
- Full-page screenshots at multiple breakpoints (mobile, tablet, desktop)
- `getComputedStyle()` sweeps for every visible element
- Interaction sweep: scroll, hover, click, focus, responsive resize
- Extracts fonts, color palette, spacing scale, border radii, shadow tokens

### Phase 2 — Foundation
- Updates `globals.css` with extracted oklch color tokens
- Installs Google Fonts or downloads font files to `/public`
- Downloads all images → `/public/images/`, videos → `/public/videos/`, favicons → `/public/seo/`
- Patches `layout.tsx` with correct metadata, og image, favicon

### Phase 3 — Component Specs
- Writes one spec file per section to `docs/research/components/`
- Each spec contains exact computed CSS values, all interaction states, content copy, asset paths, and responsive breakpoints
- No guessing — builder agents receive inline ground truth

### Phase 4 — Parallel Build
- Spawns one builder agent per section in isolated git worktrees
- Each builder reads its spec and produces a self-contained React component
- Builders run concurrently — large sites finish faster

### Phase 5 — Assembly & QA
- Merges all worktrees back to main branch
- Wires components into `src/app/page.tsx`
- Runs visual diff screenshot against the original
- Outputs side-by-side comparison to `docs/design-references/comparison.png`

---

## Project Structure

```
src/
  app/
    layout.tsx          # Root layout — fonts, metadata, globals
    page.tsx            # Assembled clone page
  components/
    ui/                 # shadcn/ui primitives (Button, Card, etc.)
    icons.tsx           # Extracted SVG icons from target site
  lib/
    utils.ts            # cn() Tailwind merge utility
  types/                # Shared TypeScript interfaces
  hooks/                # Custom React hooks

public/
  images/               # Downloaded images from target
  videos/               # Downloaded videos from target
  seo/                  # Favicons, OG images

docs/
  research/
    tokens.json         # Extracted design tokens
    components/         # Per-section spec files (*.md)
  design-references/
    comparison.png      # Visual diff output

scripts/
  sync-agent-rules.sh   # Regenerate agent instruction files from AGENTS.md
  sync-skills.mjs       # Regenerate /clone-website for all platforms

AGENTS.md               # Single source of truth for agent instructions
CLAUDE.md               # Claude Code config (imports AGENTS.md)
GEMINI.md               # Gemini CLI config (imports AGENTS.md)
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 16, App Router, React 19 |
| Styling | Tailwind CSS v4, oklch design tokens |
| Components | shadcn/ui (Radix primitives) |
| Icons | Lucide React → replaced by extracted SVGs |
| Language | TypeScript (strict mode) |

---

## Commands

```bash
npm run dev    # Start dev server at localhost:3000
npm run build  # Production build
npm run lint   # ESLint check
```

---

## Real Code Examples

### Adding an extracted component

After the pipeline runs, components land in `src/components/`. Wire them into the page:

```tsx
// src/app/page.tsx
import { HeroSection } from "@/components/hero-section";
import { FeaturesSection } from "@/components/features-section";
import { PricingSection } from "@/components/pricing-section";
import { FooterSection } from "@/components/footer-section";

export default function Page() {
  return (
    <main>
      <HeroSection />
      <FeaturesSection />
      <PricingSection />
      <FooterSection />
    </main>
  );
}
```

### Typical extracted component structure

```tsx
// src/components/hero-section.tsx
import { Button } from "@/components/ui/button";
import { cn } from "@/lib/utils";

export function HeroSection() {
  return (
    <section
      className={cn(
        "relative flex min-h-[680px] flex-col items-center justify-center",
        "bg-[oklch(0.12_0.02_264)] px-6 py-24 text-center"
      )}
    >
      <h1 className="max-w-4xl text-5xl font-bold tracking-tight text-white md:text-7xl">
        Your extracted headline here
      </h1>
      <p className="mt-6 max-w-2xl text-lg text-[oklch(0.72_0.05_264)]">
        Extracted subheading copy from the target site.
      </p>
      <div className="mt-10 flex gap-4">
        <Button size="lg" className="bg-[oklch(0.65_0.22_264)] hover:bg-[oklch(0.60_0.22_264)]">
          Primary CTA
        </Button>
        <Button size="lg" variant="outline">
          Secondary CTA
        </Button>
      </div>
      <img
        src="/images/hero-visual.webp"
        alt="Hero visual"
        className="mt-16 w-full max-w-5xl rounded-xl shadow-2xl"
      />
    </section>
  );
}
```

### Design tokens in globals.css (Tailwind v4)

```css
/* src/app/globals.css */
@import "tailwindcss";

:root {
  /* Extracted from target site via getComputedStyle() */
  --color-primary: oklch(0.65 0.22 264);
  --color-primary-hover: oklch(0.60 0.22 264);
  --color-background: oklch(0.12 0.02 264);
  --color-surface: oklch(0.16 0.02 264);
  --color-text: oklch(0.97 0.01 264);
  --color-text-muted: oklch(0.72 0.05 264);
  --color-border: oklch(0.25 0.03 264);

  --radius-sm: 0.375rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-xl: 1rem;

  --font-sans: "Inter", system-ui, sans-serif;
  --font-display: "Cal Sans", "Inter", sans-serif;
}
```

### Extracted SVG icons

```tsx
// src/components/icons.tsx
// Auto-generated by the clone pipeline — do not edit manually

export function CheckIcon({ className }: { className?: string }) {
  return (
    <svg
      className={className}
      viewBox="0 0 24 24"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
    >
      <path
        d="M20 6L9 17L4 12"
        stroke="currentColor"
        strokeWidth="2"
        strokeLinecap="round"
        strokeLinejoin="round"
      />
    </svg>
  );
}
```

### Using cn() for conditional classes

```tsx
import { cn } from "@/lib/utils";

function NavLink({
  href,
  active,
  children,
}: {
  href: string;
  active?: boolean;
  children: React.ReactNode;
}) {
  return (
    <a
      href={href}
      className={cn(
        "text-sm font-medium transition-colors",
        active
          ? "text-white"
          : "text-[oklch(0.72_0.05_264)] hover:text-white"
      )}
    >
      {children}
    </a>
  );
}
```

---

## Component Spec Format

Each file in `docs/research/components/` follows this structure:

```markdown
# Hero Section Spec

## Dimensions
- Height: 680px min, full viewport
- Padding: 96px top/bottom, 24px horizontal

## Typography
- H1: font-size 72px, font-weight 700, letter-spacing -0.02em, color #fff
- Subhead: font-size 18px, font-weight 400, color oklch(0.72 0.05 264)

## Colors
- Background: oklch(0.12 0.02 264)
- CTA button: oklch(0.65 0.22 264)

## Assets
- Hero image: /public/images/hero-visual.webp (downloaded)
- Background gradient: linear-gradient(180deg, oklch(0.12 0.02 264) 0%, oklch(0.08 0.01 264) 100%)

## States
- CTA hover: background lightens to oklch(0.70 0.22 264), scale(1.02)
- CTA active: scale(0.98)

## Responsive
- Mobile (<768px): H1 shrinks to 36px, buttons stack vertically
- Tablet (768–1024px): H1 48px, single row buttons

## Content
[exact copy extracted from target]
```

---

## Maintaining Multi-Platform Support

All agent instructions have a single source of truth. After editing, regenerate platform-specific files:

```bash
# After editing AGENTS.md
bash scripts/sync-agent-rules.sh

# After editing .claude/skills/clone-website/SKILL.md
node scripts/sync-skills.mjs
```

This regenerates configs for Claude Code, Gemini CLI, Cursor, Windsurf, Copilot, and others automatically.

---

## Supported AI Agents

| Agent | Notes |
|-------|-------|
| Claude Code | **Recommended** — Opus 4.6, use `claude --chrome` |
| Codex CLI | `codex` in project root |
| OpenCode | Reads `AGENTS.md` automatically |
| Gemini CLI | Reads `GEMINI.md` |
| Cursor | Reads `.cursor/rules/` |
| Windsurf | Reads `.windsurf/rules/` |
| GitHub Copilot | Reads `.github/copilot-instructions.md` |
| Cline, Roo Code, Continue, Aider, Amazon Q, Augment Code | All supported via `AGENTS.md` |

---

## Troubleshooting

### Clone produces wrong colors
- The pipeline uses `getComputedStyle()` — if the target site uses CSS variables with JavaScript-driven theming, run the agent with `--chrome` so it executes JS before extraction.
- Check `docs/research/tokens.json` to verify extracted values before the build phase.

### Images not loading
- All images download to `/public/images/`. If paths are wrong, check `docs/research/components/*.md` for the asset path the builder used.
- Re-run `npm run dev` — Next.js serves `/public` at root.

### Component doesn't match visually
- Open `docs/design-references/comparison.png` for the side-by-side diff.
- The spec file for that section is in `docs/research/components/`. Check computed values there and patch the component manually.
- For shadcn/ui primitive mismatches, check that you're on the correct variant: `variant="outline"` vs `variant="ghost"`.

### Parallel build conflict in worktrees
- Run `git worktree prune` to clean up stale worktrees after a failed run.
- Then re-run `/clone-website <url>` — the pipeline is idempotent.

### Font not rendering correctly
- Verify the font is listed in `src/app/layout.tsx` under `next/font/google` or as a local font in `/public/fonts/`.
- Check `globals.css` for `--font-sans` and `--font-display` token assignments.

### Build fails (TypeScript errors)
```bash
npm run lint       # Find ESLint issues
npm run build      # See full tsc errors
```
Common cause: extracted component uses an SVG prop (`stroke-width`) instead of React camelCase (`strokeWidth`). The builder should handle this, but fix manually if needed.

---

## Customization After Cloning

Once the base clone is assembled, extend it like any Next.js project:

```bash
# Add a shadcn/ui component
npx shadcn@latest add dialog

# Add a new route
mkdir src/app/pricing
touch src/app/pricing/page.tsx

# Swap a placeholder image
cp my-image.webp public/images/hero-visual.webp
```

Edit `src/app/page.tsx` to reorder, remove, or add sections. All components are standard React — no lock-in.
```
